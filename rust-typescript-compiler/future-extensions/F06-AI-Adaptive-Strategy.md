# F06 — AI 自适应编译策略

> **优先级**：🟡 中 | **复杂度**：中 | **建议 Phase**：Phase 4-5 | **工期**：~5 周

---

## 1. 概述

当前 AI 编排器是"请求-响应"模式——用户触发分析，AI 返回建议。本方案将其升级为**学习-推荐**闭环：基于历史编译数据，AI 自动学习最优的 Pass 调度策略、后端选择、优化级别，将 BlockType 变为"越用越聪明"的编译器。

### 核心理念

```
当前模式：
  [固定 Pass 序列] → 编译 → [丢弃性能数据]

自适应模式：
  [历史性能数据] → [学习最优策略] → [自适应 Pass 调度]
                                   ↓
                            [编译] → [收集新数据] → 反馈循环
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                 bt-ai 扩展（自适应策略模块）                        │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompileProfileDatabase                     │
│  │  - 编译历史数据库（基于 EventStore + SQLite）                │     │
│  │  - 每条记录：源码特征 + Pass 序列 + 后端 + 耗时 + 产出质量   │     │
│  │  - 可导出为 ML 训练数据                                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    AdaptivePassScheduler                     │     │
│  │  - 基于特征自动选择最优 Pass 序列                           │     │
│  │  - 策略：规则引擎（快速） → ML 模型（精准）                  │     │
│  │  - Pass 序列按 crate/function 粒度差异化                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    BackendSelector                           │     │
│  │  - 根据代码特征自动选择 LLVM / Cranelift                   │     │
│  │  - 小项目 → Cranelift（快编），大项目 → LLVM（深优化）     │     │
│  │  - 混合模式：热函数用 LLVM，冷函数用 Cranelift              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    LearningEngine                            │     │
│  │  - 轻量级 ML 推理引擎（ort / tract）                        │     │
│  │  - 特征提取器（从 IRModule 提取 ~50 维特征）                │     │
│  │  - 策略网络（小模型 <1MB 参数）                              │     │
│  │  - 本地训练（可选） vs 云端训练                              │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

数据流：
  CompileService
    ├── 编译前：提取特征 → AdaptivePassScheduler → 推荐 Pass 序列
    ├── 编译中：记录各 Pass 耗时 → EventStore
    ├── 编译后：记录产出质量（代码大小/性能/编译时间）→ EventStore
    └── 离线：LearningEngine 从 EventStore 读取历史 → 更新策略模型
```

### 2.2 核心类型定义

```rust
// crates/bt-ai/src/adaptive/mod.rs

// ─── 编译特征提取 ───

/// 编译特征 — 用于 Pass 调度决策的 50 维特征向量
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CompileFeatures {
    // ─── 代码结构特征 ───
    pub function_count: usize,
    pub total_instructions: usize,
    pub basic_block_count: usize,
    pub max_cfg_depth: usize,
    pub loop_count: usize,
    pub generic_function_count: usize,
    pub closure_count: usize,

    // ─── 类型特征 ───
    pub trait_object_count: usize,
    pub dyn_dispatch_count: usize,
    pub static_dispatch_ratio: f64,
    pub alloc_count: usize,

    // ─── Dialect 特征 ───
    pub dialect_opcodes: HashMap<String, usize>,

    // ─── 项目特征 ───
    pub crate_count: usize,
    pub total_deps: usize,
    pub source_lines: usize,
    pub is_ci_build: bool,
    pub target_arch: String,

    // ─── 历史特征 ───
    pub prev_compile_time_ms: u64,
    pub prev_opt_time_ms: u64,
    pub incremental_hit_ratio: f64,
}

// ─── 自适应 Pass 调度器 ───

/// Pass 序列推荐
#[derive(Debug, Clone, Serialize)]
pub struct PassSchedule {
    /// 推荐的 Pass 序列
    pub passes: Vec<PassCandidate>,
    /// 置信度
    pub confidence: f64,
    /// 预估编译时间节省
    pub estimated_savings_ms: u64,
    /// 调度策略来源
    pub source: ScheduleSource,
}

pub enum ScheduleSource {
    /// 规则引擎快速匹配
    RuleEngine,
    /// ML 模型推理
    Model,
    /// 默认策略（无历史数据时）
    Default,
}

pub struct PassCandidate {
    pub name: String,
    pub priority: usize,
    pub estimated_gain: f64,
}

/// 自适应 Pass 调度器
pub struct AdaptivePassScheduler {
    /// 规则引擎（零延迟 fallback）
    rule_engine: PassRuleEngine,
    /// ML 模型推理引擎（可选）
    ml_engine: Option<Box<dyn PassModelEngine>>,
    /// 历史数据库
    profile_db: CompileProfileDatabase,
}

impl AdaptivePassScheduler {
    /// 推荐 Pass 序列
    pub async fn recommend(&self, features: &CompileFeatures) -> PassSchedule {
        // 1. 规则引擎快速匹配
        if let Some(schedule) = self.rule_engine.match_rules(features).await {
            return schedule;
        }

        // 2. ML 模型推理（如果有历史数据）
        if let Some(ref ml) = self.ml_engine {
            if self.profile_db.has_enough_data() {
                return ml.predict(features).await;
            }
        }

        // 3. 默认策略
        PassSchedule::default()
    }

    /// 记录编译结果（用于后续学习）
    pub async fn record_outcome(
        &self,
        features: &CompileFeatures,
        schedule: &PassSchedule,
        outcome: &CompileOutcome,
    ) {
        self.profile_db.record(CompileProfile {
            features: features.clone(),
            schedule: schedule.clone(),
            outcome: outcome.clone(),
            timestamp: Utc::now(),
        }).await;
    }
}

// ─── 后端选择器 ───

/// 后端选择建议
#[derive(Debug, Clone, Serialize)]
pub struct BackendRecommendation {
    pub primary_backend: String,       // "llvm" / "cranelift"
    pub mixed_mode: Vec<MixedFunction>,// 混合模式下各函数的后端分配
}

pub struct MixedFunction {
    pub function_name: String,
    pub backend: String,
    pub reason: String,
}

/// 自适应后端选择器
pub struct BackendSelector {
    rule_engine: BackendRuleEngine,
    ml_engine: Option<Box<dyn BackendModelEngine>>,
}

impl BackendSelector {
    /// 推荐后端（函数级别粒度）
    pub async fn recommend_backend(
        &self,
        features: &CompileFeatures,
        available_backends: &[String],
    ) -> BackendRecommendation;
}

// ─── 轻量级 ML 引擎 ───

/// Pass 调度 ML 模型引擎
#[async_trait]
pub trait PassModelEngine: Send + Sync {
    fn name(&self) -> &str;
    async fn predict(&self, features: &CompileFeatures) -> PassSchedule;
    async fn train(&self, profiles: &[CompileProfile]) -> Result<(), ModelError>;
}

/// 基于 ONNX 的推理引擎（使用 tract 或 ort）
pub struct OnnxPassModel {
    model_path: PathBuf,
    session: OrtSession,
}

#[async_trait]
impl PassModelEngine for OnnxPassModel {
    fn name(&self) -> &str { "onnx-pass-model" }

    async fn predict(&self, features: &CompileFeatures) -> PassSchedule {
        let tensor = self.features_to_tensor(features);
        let output = self.session.run(tensor).await?;
        self.tensor_to_schedule(output)
    }

    async fn train(&self, profiles: &[CompileProfile]) -> Result<(), ModelError> {
        // 学习 Pass 序列与编译时间的关联
        // 可使用 LightGBM / XGBoost 训练的模型
        // 实际部署中，训练通常在云端完成，本地仅做推理
        Err(ModelError::TrainingNotSupported)
    }
}
```

### 2.3 规则引擎策略示例

```rust
// 内置的 Pass 调度规则（零延迟，无需 ML）

impl PassRuleEngine {
    fn build_rules(&mut self) {
        // 规则 1: 小型函数（<50 指令）→ 跳过内联（内联反而增加开销）
        self.add_rule(|f| f.total_instructions < 50, |f| PassSchedule::no_inlining());

        // 规则 2: 大量循环 → 优先向量化 + 循环展开
        self.add_rule(|f| f.loop_count > 10, |f| PassSchedule::heavy_loop_opt());

        // 规则 3: CI 构建 → 跳过耗时的 AI 分析
        self.add_rule(|f| f.is_ci_build, |f| PassSchedule::ci_optimized());

        // 规则 4: 大量泛型调用 → 优先单态化后内联
        self.add_rule(|f| f.generic_function_count > 50, |f| PassSchedule::monomorphize_first());

        // 规则 5: 增量编译首次 → 不做深度优化（加速编辑-编译循环）
        self.add_rule(|f| f.incremental_hit_ratio < 0.3, |f| PassSchedule::quick());
    }
}
```

### 2.4 与 CompileService 的集成点

```rust
// CompileService 扩展自适应策略

impl CompileService {
    /// 自适应编译入口
    pub async fn compile_adaptive(&self, req: CompileRequest) -> Result<CompileResponse, CompilerError> {
        // 1. 提取特征
        let features = self.extract_features(&req).await?;

        // 2. 推荐 Pass 序列
        let schedule = self.adaptive_scheduler.recommend(&features).await;

        // 3. 推荐后端
        let backend_rec = self.backend_selector.recommend_backend(
            &features,
            &self.available_backends(),
        ).await;

        // 4. 使用推荐策略执行编译
        let result = self.compile_with_schedule(req, &schedule, &backend_rec).await?;

        // 5. 记录编译结果
        self.adaptive_scheduler.record_outcome(
            &features, &schedule, &result.outcome,
        ).await;

        Ok(result)
    }
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **编译特征提取** | 自动提取 ~50 维代码结构/类型/项目特征 |
| C2 | **自适应 Pass 调度** | 基于特征自动选择最优 Pass 序列 |
| C3 | **自适应后端选择** | 根据代码特征自动选择 LLVM / Cranelift |
| C4 | **混合后端模式** | 热函数用 LLVM 深优化，冷函数用 Cranelift 快编 |
| C5 | **规则引擎** | 内置 20+ 调度规则，零延迟，离线可用 |
| C6 | **ML 模型推理** | ONNX 兼容模型，可学习历史编译数据 |
| C7 | **编译档案数据库** | 持久化历史编译记录，可导出训练数据 |
| C8 | **自适应优化** | `bt build --adaptive` 一键启用 |
| C9 | **性能 Dashboard** | 展示自适应带来的编译时间/产出质量提升 |

---

## 4. 与现有架构的集成

### 新增模块

| 模块 | 位置 | 行数估算 |
|------|------|---------|
| 自适应策略 | `bt-ai/src/adaptive/` | ~2,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-ai` | 新增 `adaptive/` 模块 | 中 |
| `bt-service` | CompileService 新增 `compile_adaptive()` 方法 | 中 |
| `bt-passes` | PassManager 支持动态 Pass 序列 | 中 |
| `bt-cli` | 新增 `--adaptive` | 低 |

### CLI 变更

```bash
# ─── 自适应模式 ───

bt build --adaptive                     # 自适应编译（自动选择策略）
bt build --adaptive=speed               # 优先编译速度
bt build --adaptive=quality             # 优先代码质量
bt build --adaptive=balanced            # 平衡模式（默认）

bt build --adaptive --backend=auto      # 自动选择后端
bt build --backend=adaptive             # 后端自适应（同 --backend=auto）

# ─── 编译档案管理 ───

bt adaptive profile                     # 显示编译档案统计
bt adaptive profile --export=train.json # 导出训练数据
bt adaptive profile --reset             # 清空历史数据

# ─── 策略可视化 ───

bt adaptive explain                     # 解释当前策略选择
bt adaptive recommend                   # 分析项目特征并推荐策略
```

---

## 5. 分阶段实施

### Phase 4 扩展（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | CompileFeatures 提取 + CompileProfileDatabase | 编译特征可记录和查询 |
| 2 | PassRuleEngine + AdaptivePassScheduler（规则引擎版本） | 基于规则的 Pass 调度 |

### Phase 5 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 3 | BackendSelector（规则版本） | 小项目自动用 Cranelift |
| 4 | ML 模型推理引擎集成（ONNX/tract） | `bt build --adaptive` 可用 |
| 5 | 混合后端模式 + 性能 Dashboard | 自适应策略可视化 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| ML 模型推理延长编译时间 | 低 | 中 | 特征提取 <5ms，模型 <1ms |
| 自适应策略导致代码质量下降 | 低 | 高 | 默认策略保底，仅学习正向改进 |
| 编译档案数据库膨胀 | 中 | 低 | 自动清理超过 30 天的旧记录 |
| 模型训练需要大量数据 | 高 | 低 | 规则引擎先可用，ML 作为增量提升 |
