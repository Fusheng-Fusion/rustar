# F11 — 编译器 LLM 协作（Compile-Time AI Collaboration）

> **优先级**：🟢 低 | **复杂度**：中 | **建议 Phase**：Phase 5+ | **工期**：~4 周

---

## 1. 概述

当前 BlockType 的 AI 集成是"单向"的——编译器产出数据，AI 分析并给出建议。本方案将其变为**双向协作**：编译器在遇到歧义或复杂的编译决策时，可以主动向 LLM 提问，根据 LLM 的回复决定下一步。这种模式更接近"编译器与开发者之间的智能助手"。

### 核心理念

```
单向模式（当前）：
  编译器 → [数据] → AI → [建议] → 开发人员

双向协作模式（新增）：
  编译器遇到歧义 → [提问] → LLM → [回答] → 编译器继续
  OR
  开发人员 ↔ [对话] ↔ AI（共享编译器上下文）
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                 bt-ai 扩展（LLM 协作模块）                          │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompileTimeQuery                         │     │
│  │  - 编译时"暂停-询问-继续"机制                              │     │
│  │  - 编译器遇到歧义时生成 LLM 查询                          │     │
│  │  - 非阻塞：查询同时编译器可继续其他独立任务                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    AmbiguityDetector                         │     │
│  │  - 检测编译过程中的"不确定点"                               │     │
│  │  - 例如：多个优化策略收益相近、类型推断存在多种可能         │     │
│  │  - 规则引擎快速匹配 + 不确定性阈值触发 LLM                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    InteractiveExplain                         │     │
│  │  - bt explain 的对话式版本                                   │     │
│  │  - 基于当前编译上下文的 AI 对话                             │     │
│  │  - 支持追问、代码示例生成、修复建议                           │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

现有基础设施：
  AIOrchestrator ← LLM 查询和对话的底层引擎
  CompilationContext ← 提供编译器内部状态的上下文
  EventStore ← 记录所有 LLM 交互（可回放、可审计）
```

### 2.2 核心类型定义

```rust
// crates/bt-ai/src/collaboration/mod.rs

// ─── 编译时查询 ───

/// 编译器向 LLM 发起的查询
pub struct CompileTimeQuery {
    pub query_id: Uuid,
    /// 查询类型
    pub kind: QueryKind,
    /// 查询上下文（编译器当前状态快照）
    pub context: CompilationContext,
    /// 问题描述
    pub question: String,
    /// 可选候选项（供 LLM 选择）
    pub choices: Option<Vec<Choice>>,
    /// 超时（默认 5s）
    pub timeout: Duration,
}

pub enum QueryKind {
    /// 优化策略选择：哪个 Pass 序列更优？
    OptimizationChoice,
    /// 类型歧义：哪种类型推断结果更合理？
    TypeAmbiguity,
    /// 代码质量：这段代码是否有更好的写法？
    CodeQuality,
    /// 错误解释：这个错误的原因和修复方法？
    ErrorExplanation,
}

pub struct Choice {
    pub id: String,
    pub description: String,
    pub estimated_impact: String,
}

/// LLM 的回复
pub struct LmResponse {
    pub query_id: Uuid,
    /// 选择的选项（如果有 choices）
    pub choice_id: Option<String>,
    /// 自然语言解释
    pub explanation: String,
    /// 置信度（0.0 - 1.0）
    pub confidence: f32,
    /// 建议的后续动作
    pub suggested_action: Option<SuggestedAction>,
}

pub enum SuggestedAction {
    /// 自动应用（高置信度）
    ApplyDirectly,
    /// 等待开发者确认
    RequestUserConfirmation,
    /// 生成代码片段
    GenerateSnippet(String),
}

// ─── 歧义检测器 ───

/// 编译歧义检测 — 决定是否以及如何向 LLM 提问
pub struct AmbiguityDetector {
    /// 零延迟规则引擎
    rule_engine: AmbiguityRuleEngine,
    /// LLM 查询服务
    query_service: Arc<CompileTimeQueryService>,
    /// 配置
    config: AmbiguityConfig,
}

pub struct AmbiguityConfig {
    /// 最小置信度阈值（低于此值触发 LLM 查询）
    pub min_confidence: f32,
    /// 是否启用"自动应用"高置信度回复
    pub auto_apply: bool,
    /// 每次编译的最大 LLM 查询次数
    pub max_queries_per_build: usize,
}

impl AmbiguityDetector {
    /// 在 Pass 执行后检查是否出现歧义
    pub async fn check_ambiguity(
        &self,
        context: &CompilationContext,
        pass_result: &PassResult,
    ) -> Option<CompileTimeQuery> {
        // 1. 规则引擎快速判断
        if self.rule_engine.matches(context, pass_result) {
            return self.rule_engine.generate_query(context, pass_result);
        }

        // 2. 不确定性评估
        if pass_result.metrics.uncertainty > self.config.min_confidence {
            let query = CompileTimeQuery {
                query_id: Uuid::new_v4(),
                kind: QueryKind::OptimizationChoice,
                context: context.clone(),
                question: format!(
                    "Pass '{}' 完成，收益不确定。多个策略可用: {:?}",
                    pass_result.pass_name,
                    pass_result.alternatives
                ),
                choices: Some(pass_result.alternatives.iter().map(|alt| Choice {
                    id: alt.name.clone(),
                    description: alt.description.clone(),
                    estimated_impact: format!("{:?}", alt.estimated_speedup),
                }).collect()),
                timeout: Duration::from_secs(5),
            };
            return Some(query);
        }

        None
    }
}

// ─── 交互式解释 ───

/// 交互式编译错误解释
pub struct InteractiveExplain {
    context_extractor: ContextExtractor,
    orchestrator: Arc<AIOrchestrator>,
    event_store: Arc<EventStore>,
}

impl InteractiveExplain {
    /// 对话式错误解释（支持追问）
    pub async fn chat(
        &self,
        task_id: Option<Uuid>,
        history: &[ChatMessage],
        new_message: &str,
    ) -> Result<String, AIError> {
        // 1. 提取编译上下文
        let context = match task_id {
            Some(id) => {
                let mut ctx = self.context_extractor.extract(id).await?;
                ctx.event_history = self.event_store.replay_task(id).await;
                ctx
            }
            None => CompilationContext::default(),
        };

        // 2. 构建提示（包含完整编译器内部状态）
        let prompt = self.build_prompt(&context, history, new_message)?;

        // 3. 调用 AI 编排器
        let req = AIAnalysisRequest {
            context,
            analysis_type: AnalysisType::Chat,
            options: AIAnalysisOptions::default(),
        };
        let suggestions = self.orchestrator.analyze(req).await?;

        // 4. 组装回复
        Ok(format_suggestions_as_reply(&suggestions, new_message))
    }

    fn build_prompt(
        &self,
        context: &CompilationContext,
        history: &[ChatMessage],
        new_message: &str,
    ) -> Result<String> {
        Ok(format!(
            "你是一个嵌入在 BlockType 编译器中的 AI 助手。\n\n\
             当前编译状态：\n\
             - 前端: {}\n\
             - 目标: {}\n\
             - 活跃 Dialect: {:?}\n\
             - 管线阶段: {}\n\
             - 诊断: {} 个\n\
             - 已注册 Pass: {} 个\n\n\
             对话历史：\n\
             {}\n\n\
             用户问题：{}\n\n\
             请基于以上编译器内部状态回答问题，可以：\n\
             1. 解释错误原因\n\
             2. 给出修复代码\n\
             3. 建议优化方案\n\
             4. 分析 IR 结构",
            context.frontend,
            context.target_triple,
            context.active_dialects,
            context.phase_metrics.keys().cloned().collect::<Vec<_>>().join(" → "),
            context.diagnostics.len(),
            context.enabled_features.len(),
            history.iter().map(|m| format!("{}: {}", m.role, m.content)).collect::<Vec<_>>().join("\n"),
            new_message,
        ))
    }
}
```

### 2.3 工作流程

```
编译管线                                  BlockType Server
───────────                                ────────────────
[Lower Pass] → [AI Pass] → [Optimize Pass]
                              │
                              ├── 检测到歧义：多个优化策略收益相近
                              │
                              ▼
                          AmbiguityDetector
                              │
                              ├── 规则引擎无法判断
                              │
                              ▼
                          CompileTimeQuery → ──────────→ LLM API
                              │                          │
                              │                     "这个函数应该先
                              │                      内联还是先 DCE？"
                              │                          │
                              │                          ▼
                              │                    LLM 回复
                              │                    "先 DCE 可以减少
                              │                     内联的工作量，
                              │                     建议先 DCE 再 Inline"
                              │                          │
                              ◀──────────────────────────┘
                              │
                              ├── 自动应用（高置信度）
                              ▼
                        继续编译 → 记录 LLM 决策到 EventStore
```

### 2.4 CLI 交互示例

```bash
$ bt explain E0502

  ── 编译器内部状态 ──
  文件: src/main.rs:3:19
  阶段: borrowck
  当前作用域: {
    x: &mut String (可变借用),
    y: &String (不可变借用, 第2行)
  }

  ── AI 对话 ──

  你: 为什么这段代码会报 E0502？

  AI: 因为你在第3行试图创建可变引用时，
      第2行创建的不可变引用 y 仍然在作用域内。
      Rust 不允许在不可变引用存活期间创建可变引用。

  你: 那怎么修复？

  AI: 有两个方案：
      方案1: 缩小 y 的作用域（在可变借用前 drop 掉）
      方案2: 如果不需要同时使用，可以用 Cell/RefCell

      要我生成具体的修复代码吗？
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **编译时 LLM 查询** | 编译器遇到歧义时主动向 LLM 提问 |
| C2 | **歧义检测引擎** | 规则引擎 + 不确定性阈值双模式 |
| C3 | **非阻塞查询** | LLM 查询期间编译器可继续其他独立任务 |
| C4 | **对话式错误解释** | `bt explain --interactive` 支持追问 |
| C5 | **上下文感知** | AI 回复自动注入当前编译器内部状态 |
| C6 | **自动应用** | 高置信度 (>0.95) 的 LLM 回复自动应用 |
| C7 | **查询审计** | 所有 LLM 交互记录到 EventStore（可回放、可审计） |
| C8 | **查询预算** | 每次编译最多 N 次 LLM 查询，防止滥用 |

---

## 4. 与现有架构的集成

### 新增模块

| 模块 | 位置 | 行数估算 |
|------|------|---------|
| 编译时查询 | `bt-ai/src/collaboration/` | ~1,500 |
| 交互式解释 | `bt-ai/src/explain/` | ~800 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-ai` | 新增 collaboration + explain 模块 | 中 |
| `bt-pass-manager` | Pass 执行后调用 AmbiguityDetector | 低（可选） |
| `bt-cli` | `bt explain` 增加 `--interactive` 模式 | 低 |

### CLI 变更

```bash
# ─── 交互式错误解释 ───

bt explain E0502                            # 标准解释（非交互）
bt explain --interactive                    # 进入对话模式
bt explain E0502 --interactive              # 带错误的对话模式

bt explain --task=<task_id>                 # 基于某次编译的上下文解释

# ─── 编译时 AI 协作 ───

bt build --ai=collaborate                   # 启用编译时 AI 协作
bt build --ai=collaborate --max-ai-queries=3  # 限制每次编译最多 3 次 AI 查询
bt build --ai=collaborate --auto-apply      # 自动应用高置信度建议

bt build --ai=manual                        # 遇到歧义时等待用户输入
```

---

## 5. 分阶段实施

### Phase 5 扩展（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | CompileTimeQuery + AmbiguityDetector + LLM 查询 | 编译器可在 Pass 执行后查询 LLM |
| 2 | 查询审计 + 预算管理 + EventStore 记录 | LLM 交互可回放和审计 |

### 扩展阶段（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 3 | InteractiveExplain + 对话式错误解释 | `bt explain --interactive` |
| 4 | 自动应用 + CLI 参数 | `bt build --ai=collaborate` |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| LLM 延迟阻塞编译 | 中 | 中 | 非阻塞 + 超时机制（默认 5s 超时后使用默认策略） |
| LLM 给出错误建议 | 中 | 高 | 仅 high confidence 自动应用，其他需确认 |
| 编译成本增加（LLM 调用费）| 中 | 中 | 查询预算限制 + 规则引擎预过滤减少不必要查询 |
