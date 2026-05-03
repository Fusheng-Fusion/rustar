# F01 — 分布式编译缓存

> **优先级**：🔴 高 | **复杂度**：高 | **建议 Phase**：Phase 2-3 | **工期**：~6 周

---

## 1. 概述

当前 BlockType 的 Salsa 查询引擎（`bt-query`）仅支持本地增量编译。对于 CI/CD 场景和多开发者团队，编译结果的**跨机器共享**可以大幅提升编译效率。本方案在增量编译之上构建**分布式缓存层**，使编译产物在团队/CI 间自动共享。

### 核心思路

```
本地编译（现有）：  源码 → [bt-query 本地缓存] → 编译 → 产物
分布式编译（新增）： 源码 → [bt-query 本地缓存] → [远程缓存] → 编译 → 产物
                                            ↑ 跨机器共享
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     bt-build-cache crate                           │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                  CacheBackend trait                         │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │     │
│  │  │ LocalDisk│  │  Redis   │  │   S3     │  ← 可插拔后端   │     │
│  │  │ (默认)   │  │(集群共享)│  │(远程CI)  │                  │     │
│  │  └──────────┘  └──────────┘  └──────────┘                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                  ContentHashCache                           │     │
│  │  - 内容寻址缓存（基于源码哈希 + 编译参数）                     │     │
│  │  - 缓存层次：L1(内存) → L2(本地磁盘) → L3(远程)             │     │
│  │  - 缓存失效策略：TTL + LRU + 显式失效                        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                  RemoteExecutionClient                      │     │
│  │  - Remote Execution API (REAPI) 兼容                      │     │
│  │  - BuildBarn / Native Build 后端                           │     │
│  │  - 流式日志 + 心跳                                         │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

与 bt-query 集成：
  bt-query (Salsa 风格)
     ├── [本地] Arc<RwLock<FxHashMap<QueryKey, QueryValue>>>
     ├── [远程] Arc<RwLock<ContentHashCache>>  ← 新增
     │            ├── query.on_miss() → 检查远程缓存
     │            ├── query.on_hit() → 返回缓存结果
     │            └── query.on_compute() → 写入远程缓存
     └── [透明] 缓存逻辑对 Pass 开发者不可见
```

### 2.2 核心类型定义

```rust
// crates/bt-build-cache/src/lib.rs

// ─── 缓存后端 trait ───

/// 可插拔缓存后端
#[async_trait]
pub trait CacheBackend: Send + Sync {
    /// 后端名称
    fn name(&self) -> &str;

    /// 检查缓存是否存在
    async fn contains(&self, key: &CacheKey) -> Result<bool, CacheError>;

    /// 读取缓存
    async fn get(&self, key: &CacheKey) -> Result<Option<CacheEntry>, CacheError>;

    /// 写入缓存
    async fn put(&self, key: CacheKey, entry: CacheEntry) -> Result<(), CacheError>;

    /// 批量读取
    async fn get_batch(&self, keys: &[CacheKey]) -> Result<Vec<Option<CacheEntry>>, CacheError>;

    /// 显式失效
    async fn invalidate(&self, key: &CacheKey) -> Result<(), CacheError>;
}

// ─── 缓存键 ───

/// 缓存键 — 基于内容哈希的可复现键
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct CacheKey {
    /// 源码内容哈希（SHA-256）
    pub source_hash: String,
    /// 编译参数哈希（优化级别、target、feature 组合等）
    pub config_hash: String,
    /// 依赖哈希（所有直接依赖的缓存键的 Merkle 树根）
    pub deps_hash: String,
    /// 缓存层级：crate / function / module
    pub granularity: CacheGranularity,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum CacheGranularity {
    /// crate 级别（最大粒度，命中率高）
    Crate,
    /// 函数级别（最小粒度，复用率高）
    Function,
}

// ─── 缓存条目 ───

/// 缓存条目
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CacheEntry {
    pub key: CacheKey,
    pub value: Vec<u8>,              // 序列化的编译结果
    pub kind: CacheEntryKind,
    pub created_at: DateTime<Utc>,
    pub ttl_seconds: Option<u64>,     // None = 永不过期
    pub size_bytes: u64,
    pub trace_id: Option<String>,     // 用于可观测性追踪
}

pub enum CacheEntryKind {
    /// 完整编译产物（.o / .rlib）
    Artifact,
    /// 查询结果缓存
    QueryResult,
    /// IR 模块缓存
    IRModule,
}

// ─── 缓存统计 ───

/// 缓存命中统计
#[derive(Debug, Clone, Serialize)]
pub struct CacheStats {
    pub hits: u64,
    pub misses: u64,
    pub hit_ratio: f64,
    pub bytes_saved: u64,
    pub time_saved_ms: u64,
    pub backend_name: String,
}

// ─── 远程执行客户端 ───

/// 远程执行配置
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RemoteExecutionConfig {
    pub endpoint: String,              // REAPI 端点
    pub platform: HashMap<String, String>, // 执行平台约束
    pub priority: i32,
    pub timeout_seconds: u64,
    pub max_parallel: usize,
}

/// 远程执行客户端
pub struct RemoteExecutionClient {
    config: RemoteExecutionConfig,
    /// HTTP 客户端（用于 REAPI 通信）
    client: reqwest::Client,
}

impl RemoteExecutionClient {
    /// 提交远程执行请求
    pub async fn execute(&self, action: RemoteAction) -> Result<RemoteActionResult>;

    /// 获取执行结果（轮询或 WebSocket 推送）
    pub async fn wait_result(&self, action_id: &str) -> Result<RemoteActionResult>;
}
```

### 2.3 多层缓存架构

```
请求缓存
    │
    ▼
L1: MemoryCache (Rust HashMap, 无序列化)
    ├── 命中 → 立即返回
    └── 未命中
         │
         ▼
    L2: LocalDiskCache (bincode 序列化, SSD)
         ├── 命中 → 反序列化 → 写入 L1 → 返回
         └── 未命中
              │
              ▼
         L3: RemoteCache (Redis / S3 / REAPI)
              ├── 命中 → 下载 → 写入 L2 + L1 → 返回
              └── 未命中
                   │
                   ▼
              执行编译 → 写入 L3 → L2 → L1 → 返回
```

### 2.4 与 bt-query 的集成点

```rust
// bt-query 的 SalsaQueryEngine 扩展

impl SalsaQueryEngine {
    /// 附加分布式缓存后端（可选）
    pub fn attach_cache(&mut self, cache: Arc<dyn CacheBackend>) {
        self.cache_backend = Some(cache);
    }

    /// 查询入口（增加缓存穿透逻辑）
    pub async fn query<Q: Query>(&self, key: Q) -> Result<Q::Value, QueryError> {
        let cache_key = self.make_cache_key(&key);

        // 1. 检查本地缓存（现有逻辑）
        if let Some(value) = self.local_cache.get(&cache_key) {
            self.record_hit(Local);
            return Ok(value);
        }

        // 2. 检查远程缓存（新增）
        if let Some(ref remote) = self.cache_backend {
            if let Some(cached) = remote.get(&cache_key).await? {
                let value: Q::Value = deserialize(&cached.value)?;
                self.local_cache.insert(cache_key.clone(), value.clone());
                self.record_hit(Remote);
                return Ok(value);
            }
        }

        // 3. 执行计算（现有逻辑）
        let value = self.compute(key).await?;

        // 4. 写入远程缓存（新增）
        if let Some(ref remote) = self.cache_backend {
            let entry = CacheEntry {
                key: cache_key.clone(),
                value: serialize(&value)?,
                kind: CacheEntryKind::QueryResult,
                created_at: Utc::now(),
                ttl_seconds: Some(3600),
                size_bytes: 0,
                trace_id: None,
            };
            let _ = remote.put(cache_key, entry).await;
        }

        Ok(value)
    }
}
```

### 2.5 缓存键 Merkle 树

```
CacheKey (crate 级别)
 ├── source_hash: sha256(source_code)
 ├── config_hash: sha256(opt_level + target + features + ...)
 └── deps_hash: sha256(
         dep1_cache_key.hash +
         dep2_cache_key.hash +
         ...
     )

范例：
  Crate A：修改了源码 → source_hash 变化 → 缓存失效
  Crate B：依赖 crate A 但自身未修改 → deps_hash 变化 → 缓存失效
  Crate C：未修改且依赖未变 → 缓存命中
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **缓存分层** | L1(内存) → L2(本地磁盘) → L3(Redis/S3)，自动降级 |
| C2 | **内容寻址** | 基于 SHA-256 内容哈希，保证缓存一致性 |
| C3 | **Merkle 依赖树** | 依赖链变更自动级联失效 |
| C4 | **可插拔后端** | LocalDisk / Redis / S3 / REAPI，通过 CacheBackend trait 扩展 |
| C5 | **远程执行** | 支持 REAPI 标准，可将编译任务提交到远程集群 |
| C6 | **缓存预热** | CI 可以预先生成常用 crate 的缓存 |
| C7 | **缓存统计** | 命中率/节省时间/节省带宽，通过 EventStore 记录 |
| C8 | **调试模式** | `bt build --cache=debug` 显示缓存决策日志 |
| C9 | **缓存冻结** | 发布版本时可以冻结缓存，确保可重现构建 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-build-cache` | 缓存抽象 + 多层缓存 + 远程执行客户端 | ~3,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-query` | 新增 `attach_cache()` + 缓存穿透逻辑 | 低（可选特性） |
| `bt-cargo` | 缓存键生成 + Merkle 树构建 | 中 |
| `bt-cli` | 新增 `--cache` 参数 | 低 |

### CLI 变更

```bash
# ─── 缓存操作 ───

bt build --cache=local                  # 仅本地缓存（默认）
bt build --cache=redis                  # 本地 + Redis 集群缓存
bt build --cache=s3                     # 本地 + S3 远程缓存
bt build --cache=off                    # 禁用缓存（调试用）

bt build --cache-warmup                 # 缓存预热（仅计算，不编译）
bt build --cache-freeze                 # 冻结缓存（发布模式）

bt cache stats                          # 显示缓存统计信息
bt cache clear                          # 清空本地缓存
bt cache invalidate <crate>             # 显式失效某个 crate 的缓存

# ─── 远程执行 ───

bt build --remote-exec=buildbarn:8080   # 远程执行
bt build --remote-platform=linux/amd64  # 远程执行平台约束
```

---

## 5. 分阶段实施

### Phase 2 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-build-cache` crate: CacheBackend trait + LocalDisk 实现 | 本地磁盘缓存可用 |
| 2 | Merkle 依赖树 + 自动失效 | 修改依赖后缓存正确失效 |
| 3 | `bt-query` 集成 + CLI `--cache` 参数 | `bt build --cache=local` 完整工作 |

### Phase 3 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 4 | Redis 后端实现 | 团队共享缓存（Redis） |
| 5 | S3 后端实现 | CI 远程缓存（S3） |
| 6 | 缓存统计 + EventStore 记录 + Dashboard 展示 | 缓存命中率可视化 |

### Phase 5 扩展（可选）

| 周 | 任务 | MVP |
|---|------|-----|
| - | 远程执行（REAPI）客户端 | `bt build --remote-exec` |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 缓存键冲突（不同编译参数命中相同键） | 低 | 高 | `config_hash` 包含全量参数 |
| 缓存膨胀（占用过多磁盘） | 中 | 低 | LRU + 设置缓存上限 |
| 远程缓存网络延迟 | 中 | 中 | L1+L2 本地缓存吸收大部分请求 |
| S3 缓存序列化/反序列化开销 | 低 | 中 | bincode 二进制 + 异步批量读写 |
