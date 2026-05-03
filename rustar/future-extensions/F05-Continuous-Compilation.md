# F05 — 持续编译（Continuous Compilation）

> **优先级**：🔴 高 | **复杂度**：中 | **建议 Phase**：Phase 4 | **工期**：~4 周

---

## 1. 概述

当前 BlockType 是"请求-响应"式编译——用户执行 `bt build` 后等待输出完毕。但在 IDE 时代，开发者期望的是**文件保存即增量分析**、**诊断实时推送**、**HMR（热模块替换）** 的体验。本方案借鉴 VSCode 的 TypeScript 体验，将 BlockType 变为**持续运行的编译服务**。

### 核心体验

```
IDE 中保存文件
    │
    ▼
bt watch 检测变更
    ├── Salsa 增量查询 → 仅重编译受影响部分
    ├── 诊断结果 → WebSocket → IDE (LSP) / Dashboard
    ├── HMR 增量更新包 → 开发服务器 → 浏览器/进程热更新
    └── 编译时间线 → EventStore → 可视化
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                       bt-watch crate                               │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    FileWatcher                               │     │
│  │  - notify crate（跨平台文件系统监控）                         │     │
│  │  - 防抖/批处理：100ms 内多个变更合并为一次                   │     │
│  │  - 文件变更 → 触发 Salsa 增量查询                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    IncrementalPipeline                        │     │
│  │  - 在 CompileService 之上封装增量管线                       │     │
│  │  - 每次变更只重编译受影响的 crate/function                  │     │
│  │  - 编译 → 诊断 → WebSocket 推送（<200ms）                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    HMR (Hot Module Replacement)               │     │
│  │  - 生成增量更新包（patch 格式）                               │     │
│  │  - 通过 WebSocket 推送到开发服务器                           │     │
│  │  - 支持 Rust WASM 目标的热替换                               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompileTimeline                            │     │
│  │  - 编译时间线记录（每个文件的变更历史）                       │     │
│  │  - 异常编译检测（编译时间突增）                              │     │
│  │  - Dashboard 实时可视化                                      │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

与现有基础设施的集成：
  bt-event-store ← 记录每个文件的编译事件
  bt-telemetry   ← 记录每个变更的 Span
  bt-api (WS)    ← 推送诊断结果
  bt-query       ← Salsa 增量查询（核心依赖）
```

### 2.2 核心类型定义

```rust
// crates/bt-watch/src/lib.rs

// ─── 文件监视器 ───

/// 持续编译的主控制器
pub struct WatchService {
    /// 文件系统监视器
    watcher: FileWatcher,
    /// 增量编译管线
    pipeline: IncrementalPipeline,
    /// HMR 引擎
    hmr: Option<HmrEngine>,
    /// 已启动的编译服务器（用于 HMR）
    dev_server: Option<DevServer>,
    /// 配置
    config: WatchConfig,
}

/// 持续编译配置
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WatchConfig {
    /// 文件变更防抖间隔（默认 100ms）
    pub debounce_ms: u64,
    /// 是否启用 HMR
    pub enable_hmr: bool,
    /// HMR 目标（wasm / native debug）
    pub hmr_target: HmrTarget,
    /// 开发服务器端口（HMR 推送用）
    pub dev_server_port: u16,
    /// 增量编译超时（默认 30s）
    pub incremental_timeout: Duration,
    /// 诊断推送目标
    pub diagnostic_targets: Vec<DiagnosticTarget>,
}

pub enum HmrTarget {
    /// WASM 目标（浏览器 HMR）
    Wasm,
    /// Native 目标（通过 LD_PRELOAD / ptrace 热替换）
    NativeDebug,
}

pub enum DiagnosticTarget {
    /// 推送到 LSP 客户端
    Lsp,
    /// 推送到 Web Dashboard
    Dashboard,
    /// 同时推送
    Both,
}

// ─── 增量编译管线 ───

/// 增量编译管线 — 在 CompileService 之上封装
pub struct IncrementalPipeline {
    /// 底层编译服务
    compile_service: Arc<CompileService>,
    /// Salsa 查询引擎
    query_engine: Arc<SalsaQueryEngine>,
    /// 事件存储
    event_store: Arc<EventStore>,
}

impl IncrementalPipeline {
    /// 处理文件变更事件
    pub async fn on_file_changed(&self, changes: Vec<FileChange>) -> IncrementalResult {
        let timer = Instant::now();

        // 1. 计算受影响的文件和 crate
        let affected = self.query_engine.compute_affected(changes).await?;

        // 2. 仅重编译受影响的部分
        let compiled = Vec::new();
        for crate_info in &affected.crates {
            let result = self.compile_service.compile_crate(crate_info, &affected.deps).await?;
            compiled.push(result);
        }

        // 3. 提取诊断
        let diagnostics = compiled.iter()
            .flat_map(|c| c.diagnostics.iter())
            .cloned()
            .collect();

        // 4. 生成 HMR 更新（如果启用）
        let hmr_update = if let Some(hmr) = &self.hmr {
            Some(hmr.generate_update(&compiled).await?)
        } else {
            None
        };

        // 5. 记录事件
        self.event_store.append(CompilerEvent::IncrementalCompilation {
            duration_ms: timer.elapsed().as_millis() as u64,
            affected_files: affected.files.len(),
            affected_crates: affected.crates.len(),
            diagnostics_count: diagnostics.len(),
        }).await;

        Ok(IncrementalResult {
            duration_ms: timer.elapsed().as_millis() as u64,
            affected_files: affected.files,
            diagnostics,
            hmr_update,
        })
    }
}

// ─── HMR 引擎 ───

/// 热模块替换引擎
pub struct HmrEngine {
    /// 开发服务器连接
    dev_server: DevServerConnection,
}

impl HmrEngine {
    /// 生成增量更新包
    pub async fn generate_update(
        &self,
        compiled: &[BridgeOutput],
    ) -> Result<HmrUpdate, HmrError>;

    /// 推送更新到开发服务器
    pub async fn push_update(&self, update: HmrUpdate) -> Result<(), HmrError>;
}

/// HMR 更新包
#[derive(Debug, Clone, Serialize)]
pub struct HmrUpdate {
    pub patch_id: String,
    pub modules: Vec<HmrModule>,
    pub replaces: HashMap<String, HmrFunction>,
    pub metadata: HashMap<String, String>,
}

pub struct HmrModule {
    pub name: String,
    pub wasm_bytes: Vec<u8>,
    pub exports: Vec<String>,
}

// ─── 编译时间线 ───

/// 编译时间线记录
#[derive(Debug, Clone, Serialize)]
pub struct CompileTimeline {
    pub project: String,
    pub sessions: Vec<WatchSession>,
}

pub struct WatchSession {
    pub started_at: DateTime<Utc>,
    pub ended_at: Option<DateTime<Utc>>,
    pub total_increments: u64,
    pub avg_response_ms: f64,
    pub files: Vec<FileCompileRecord>,
}

pub struct FileCompileRecord {
    pub path: String,
    pub changes: u64,
    pub avg_time_ms: f64,
    pub errors: u64,
    pub warnings: u64,
}
```

### 2.3 工作流程

```
用户操作                   bt watch 服务                  IDE/Dashboard
─────────                 ─────────────                  ────────────
                          启动 WatchService
                           ├── FileWatcher 注册监听
                           ├── 启动 DevServer (HMR)
                           └── 建立 WebSocket 连接
                                │
用户编辑 src/lib.rs ──────→ 检测文件变更
                                │
                           debounce 100ms
                                │
                           Salsa: 计算受影响范围
                                ├── lib.rs 自身
                                └── 依赖 lib.rs 的 mod tests
                                │
                           增量编译 lib.rs
                                │
                           生成诊断
                                │
                               ├─→ WebSocket 推送诊断 ──→ LSP 更新
                               │
                                │
                           生成 HMR 更新包（如启用）
                               │
                               └─→ DevServer 推送 ──→ 浏览器热更新
```

### 2.4 与 LSP 的集成

```rust
// bt-watch 与 LSP 共享诊断推送通道

impl LspAdapter {
    /// 接收来自 bt-watch 的增量诊断推送
    pub fn on_incremental_diagnostics(
        &self,
        diagnostics: Vec<Diagnostic>,
    ) {
        // 按文件分组
        let by_file: HashMap<Url, Vec<Diagnostic>> = diagnostics.into_iter()
            .fold(HashMap::new(), |mut acc, diag| {
                let url = diag.url();
                acc.entry(url).or_default().push(diag);
                acc
            });

        // 通过 LSP protocol 推送 textDocument/publishDiagnostics
        for (url, diags) in by_file {
            self.send_notification("textDocument/publishDiagnostics", serde_json::json!({
                "uri": url,
                "diagnostics": diags,
            }));
        }
    }
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **文件系统监控** | 基于 notify crate，支持多平台（inotify/FSEvents/ReadDirectoryChanges） |
| C2 | **防抖批处理** | 100ms 窗口内合并多次变更，避免高频触发 |
| C3 | **增量编译** | Salsa 依赖追踪，每次变更只重编译受影响 crate |
| C4 | **诊断实时推送** | 编译结果通过 WebSocket 即时推送到 LSP / Dashboard |
| C5 | **HMR 热替换** | WASM 目标支持浏览器热更新，Native 目标支持调试热替换 |
| C6 | **开发服务器** | 内置轻量 DevServer，接收 HMR 更新包并转发 |
| C7 | **编译时间线** | 每个文件的编译耗时/错误数/变更频率追踪 |
| C8 | **异常检测** | 编译时间突增自动告警（如引入大型泛型） |
| C9 | **编译策略建议** | 基于历史数据建议优化编译策略 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-watch` | 文件监视 + 增量管线 + HMR + 时间线 | ~3,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-api` | 新增 `/ws/v1/watch` WebSocket 端点 + DevServer | 中 |
| `bt-service` | CompileService 增加增量编译入口 | 中（新增 `compile_incremental` 方法） |
| `bt-query` | 暴露 `compute_affected(changes)` 方法 | 低 |
| `bt-cli` | 新增 `watch` | 低 |

### CLI 变更

```bash
# ─── 持续编译 —──

bt watch                               # 启动持续编译服务
bt watch --port=3000                    # 指定开发服务器端口
bt watch --no-hmr                      # 禁用 HMR（仅增量编译）

# ─── 与 bt build 的搭配 ───

bt build --watch                       # 编译 + 持续监控
bt build --watch --open                # 编译 + 监控 + 打开 Dashboard
```

---

## 5. 分阶段实施

### Phase 4 扩展（4 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-watch` crate: FileWatcher + debounce + 文件变更检测 | `bt watch` 显示文件变更日志 |
| 2 | IncrementalPipeline: Salsa 增量查询集成 + 增量编译 | 每次保存触发增量编译 |
| 3 | WebSocket 诊断推送 + LSP 集成 | 诊断实时推送到 VS Code |
| 4 | 编译时间线 + Dashboard 可视化 | Dashboard 显示编译历史 |

### Phase 5 扩展（可选）

| 周 | 任务 | MVP |
|---|------|-----|
| - | HMR 引擎（WASM 目标） | 浏览器 WASM 热替换 |
| - | 异常编译检测 | 编译时间突增告警 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 文件系统监控在大型项目中事件风暴 | 高 | 中 | 防抖 + 批处理 + 可配置去抖间隔 |
| Salsa 增量查询在复杂依赖中耗时 | 中 | 中 | 依赖图缓存 + 异步查询 |
| HMR 在 Native 目标上实现困难 | 高 | 高 | 首版仅支持 WASM 目标 HMR |
| WebSocket 推送对 LSP 协议兼容性 | 低 | 中 | 严格遵循 LSP 3.17 规范 |
