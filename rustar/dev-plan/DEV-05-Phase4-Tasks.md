# Phase 4 — AI 增强 + 可观测性 + LSP

> **目标**：AI 辅助优化 + IDE 集成 + 全链路可观测 + 三后端
> **估计**：~10 周（50 天）| 4 个 Sprint | 15 个 Task
> **里程碑**：AI 辅助优化 + 三后端 + IDE 集成 + 实时监控全部可用

---

## Sprint 4-1: bt-ai AI 编排器（2 周）

### T-04-01: bt-ai — AIProvider trait + 多 Provider 实现

- **ID**: `T-04-01`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-04-1

**描述**：实现 AIProvider trait 和 OpenAI/Anthropic/本地规则引擎三个 provider。

**产出文件**：
- `crates/bt-ai/src/{lib.rs, provider.rs, providers/openai.rs, providers/anthropic.rs, providers/rule_engine.rs}`

**设计规格**（REF）：
- `07-AI-Native.md §5.2` — AIOrchestrator
- `07-AI-Native.md §5.3` — AIProvider trait

**前置依赖**（DEP）：
- `T-00-02`：bt-core

**验收标准**（TST）：
- [ ] `AIProvider` trait: async analyze/stream/fallback
- [ ] `OpenAIProvider`: HTTP 调用 GPT-4/Claude
- [ ] `AnthropicProvider`: HTTP 调用 Claude
- [ ] `RuleBasedProvider`: 零延迟静态规则分析
- [ ] 每个分析结果含 suggestions/confidence/tokens_used
- [ ] `cargo test -p bt-ai` 通过

---

### T-04-02: bt-ai — AIOrchestrator + fallback 链 + TokenBudget

- **ID**: `T-04-02`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P0
- **Sprint**: S-04-1

**描述**：实现 AI 编排器，管理多 Provider、fallback 链、Token 预算。

**产出文件**：
- `crates/bt-ai/src/orchestrator.rs`
- `crates/bt-ai/src/budget.rs`
- `crates/bt-ai/src/cache.rs`

**设计规格**（REF）：
- `07-AI-Native.md §5.2` — AIOrchestrator
- `05-Unified-API.md §2.10.6` — TokenBudget

**前置依赖**（DEP）：
- `T-04-01`

**验收标准**（TST）：
- [ ] `AIOrchestrator::new(providers, budget, cache)`
- [ ] `analyze(module)` → 遍历 provider 链，首个成功返回
- [ ] `analyze_stream(module)` → SSE 流式输出
- [ ] Fallback: OpenAI 失败 → Anthropic → RuleBased
- [ ] `TokenBudget`: per_request/daily/hourly 限制
- [ ] `ExhaustionPolicy`: Reject / FallbackToRuleBased
- [ ] `AIResultCache`: 按 IR hash 缓存分析结果
- [ ] 预算耗尽自动降级

---

### T-04-03: bt-ai — AIAction + 自动变换

- **ID**: `T-04-03`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-04-1

**描述**：实现 AI 分析结果的 Action 模型和自动变换。

**产出文件**：
- `crates/bt-ai/src/action.rs`

**设计规格**（REF）：
- `07-AI-Native.md §5.5`

**前置依赖**（DEP）：
- `T-04-02`

**验收标准**（TST）：
- [ ] `AIAction` 枚举：Suggestion/AutoApply/RequireReview
- [ ] `AutoApply` 自动应用高置信度变换
- [ ] `RequireReview` 低置信度需要人工确认
- [ ] AI Pass 集成到编译管线

---

## Sprint 4-2: AI Pass 集成 + 可观测性完善（2 周）

### T-04-04: AI Pass 集成到 Rust 编译管线

- **ID**: `T-04-04`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-04-2

**描述**：将 AI 分析集成到编译管线的三个插槽（PostAnalysis/PostLower/PostCodegen）。

**产出文件**：
- `crates/bt-service/src/ai_pass.rs`
- `crates/bt-service/src/compile_service.rs` — 更新

**设计规格**（REF）：
- `07-AI-Native.md §5.1` — AI 插槽

**前置依赖**（DEP）：
- `T-04-02`, `T-01-08`：bt-passes

**验收标准**（TST）：
- [ ] `bt build --ai=auto` 启用 AI 辅助
- [ ] PostAnalysis: 类型推断辅助 + 反模式检测
- [ ] PostLower: Pass 推荐 + 热点预测
- [ ] PostCodegen: 代码质量评估
- [ ] AI 结果记录到 EventStore
- [ ] `bt build --ai=off` 不调用 AI（默认）

---

### T-04-05: bt-telemetry — OpenTelemetry 全链路追踪

- **ID**: `T-04-05`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P0
- **Sprint**: S-04-2

**描述**：完善全链路追踪，每个编译阶段有 Span。

**产出文件**：
- `crates/bt-telemetry/src/tracer.rs`
- `crates/bt-telemetry/src/pipeline_node.rs`

**设计规格**（REF）：
- `06-Observability.md §4.1`

**前置依赖**（DEP）：
- `T-00-06`

**验收标准**（TST）：
- [ ] 每个编译阶段（lex/parse/analyze/lower/optimize/codegen）有 Span
- [ ] Span 属性含阶段统计（tokens/nodes/instructions 等）
- [ ] Trace ID 贯穿整个编译流程
- [ ] 可导出 Jaeger/Zipkin 格式

---

### T-04-06: bt-api — 可观测性 API + WebSocket

- **ID**: `T-04-06`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-04-2

**描述**：添加可观测性和 WebSocket 端点。

**产出文件**：
- `crates/bt-api/src/handlers/telemetry.rs`
- `crates/bt-api/src/handlers/websocket.rs`

**设计规格**（REF）：
- `05-Unified-API.md §2.6` — 可观测性 API
- `05-Unified-API.md §2.7` — WebSocket

**前置依赖**（DEP）：
- `T-00-07`, `T-04-05`

**验收标准**（TST）：
- [ ] `GET /api/v1/telemetry/pipeline|traces|metrics|logs|events`
- [ ] `GET /api/v1/telemetry/events/{task_id}` 按任务回放
- [ ] `WS /ws/v1/pipeline|diagnostics|progress` 实时推送

---

## Sprint 4-3: LSP + Cranelift 后端（2 周）

### T-04-07: bt-api — LSP 适配器

- **ID**: `T-04-07`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-04-3

**描述**：内置 LSP 适配器，支持 VS Code 基本代码补全和诊断。

**产出文件**：
- `crates/bt-api/src/lsp/{mod.rs, server.rs, handler.rs}`

**设计规格**（REF）：
- `05-Unified-API.md` — LSP stdin/stdout

**前置依赖**（DEP）：
- `T-00-07`

**验收标准**（TST）：
- [ ] LSP stdin/stdout 通信
- [ ] textDocument/didOpen + didChange + diagnostics
- [ ] textDocument/completion 基本补全
- [ ] textDocument/hover 类型信息
- [ ] VS Code 可连接并显示诊断

---

### T-04-08: bt-backend-cranelift — Cranelift 后端

- **ID**: `T-04-08`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-04-3

**描述**：实现 Cranelift 后端，支持 WASM 目标。

**产出文件**：
- `crates/bt-backend-cranelift/src/{lib.rs, backend.rs, codegen.rs, type_conv.rs}`

**前置依赖**（DEP）：
- `T-00-15`：bt-backend-common
- `T-01-01`：完整 IRType

**验收标准**（TST）：
- [ ] `CraneliftBackend` impl `Backend`
- [ ] `emit_object()` 支持 WASM 目标
- [ ] IRType → Cranelift 类型映射
- [ ] IRInstruction → Cranelift IR 映射
- [ ] `cargo test -p bt-backend-cranelift` 通过

---

### T-04-09: bt-backend-rustc — rustc 原生 LLVM 后端（新增）

- **ID**: `T-04-09`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-04-3

**描述**：实现 bt-backend-rustc，支持委托 rustc 原生 LLVM 后端进行代码生成。
该后端不实现 Backend trait（不消费 IRModule），而是在 CompileService 中
判断 backend=="rustc" 时不中断 rustc 编译，让其用自己的 LLVM 后端完成 codegen。

**产出文件**：
- `crates/bt-backend-rustc/src/{lib.rs, selector.rs, config.rs}`

**前置依赖**（DEP）：
- `T-00-04`：bt-service（CompileService 分叉逻辑）
- `T-00-09`：bt-rustc-bridge（after_analysis 分叉）

**验收标准**（TST）：
- [ ] `bt build --backend=rustc` 使用 rustc 原生 LLVM
- [ ] `bt build --backend=llvm` 使用 inkwell（默认，不变）
- [ ] `bt build --backend=auto` SmartBackendSelector 自动选择
- [ ] rustc 后端路径保留所有 BlockType 分析能力（AI、可观测、EventStore）
- [ ] `SmartBackendSelector` 检测 MIR body 中的不可映射构造 → 自动回退 rustc
- [ ] 测试：含 `asm!()` 的 crate 在 `--backend=auto` 下自动走 rustc 后端

---

## Sprint 4-4: Dashboard + 回归（2 周）

### T-04-10: Dashboard Web 监控仪表盘 + CLI 命令

- **ID**: `T-04-10`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P2
- **Sprint**: S-04-4

**描述**：Web 监控仪表盘 + `bt dashboard` / `bt trace` CLI 命令。

**产出文件**：
- `dashboard/` — React/Vue SPA
- `crates/bt-cli/src/commands/dashboard.rs` — `bt dashboard` 命令
- `crates/bt-cli/src/commands/trace.rs` — `bt trace <task_id>` 命令

**设计规格**（REF）：
- `06-Observability.md §4.5` — 监控仪表盘
- `08-Rust-Ecosystem-Integration.md §8.7` — CLI `bt dashboard` / `bt trace`

**前置依赖**（DEP）：
- `T-04-06`：WebSocket 端点

**验收标准**（TST）：
- [ ] 实时显示编译进度（WebSocket）
- [ ] 显示性能指标图表
- [ ] 事件历史浏览
- [ ] 管线节点状态可视化
- [ ] `bt dashboard` 启动 Web 仪表盘
- [ ] `bt trace <task_id>` 查看编译 Trace

---

### T-04-11: SSE 流式响应

- **ID**: `T-04-11`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P1
- **Sprint**: S-04-4

**描述**：AI 分析端点支持 SSE 流式输出。

**产出文件**：
- `crates/bt-api/src/handlers/sse.rs`

**前置依赖**（DEP）：
- `T-03-10`

**验收标准**（TST）：
- [ ] `POST /api/v1/ai/analyze` 支持 `Accept: text/event-stream`
- [ ] 逐 token 推送分析结果
- [ ] 客户端可实时显示 AI 分析

---

### T-04-12: bt-api — IR Diff API 实现

- **ID**: `T-04-12`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-04-4

**描述**：实现 IR Diff API 的三种格式输出。

**产出文件**：
- `crates/bt-api/src/handlers/ir_diff.rs`

**设计规格**（REF）：
- `05-Unified-API.md §2.3.1`

**前置依赖**（DEP）：
- `T-01-09`

**验收标准**（TST）：
- [ ] text 格式：统一 diff 文本
- [ ] structured 格式：按 IR 节点的结构化 diff
- [ ] visual 格式：Dashboard 可视化数据

---

### T-04-13: 端到端测试 — AI + 三后端

- **ID**: `T-04-13`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-04-4

**验收标准**（TST）：
- [ ] `bt build --ai=auto` 生成 AI 建议
- [ ] LLVM x86_64 + AArch64 目标编译
- [ ] Cranelift WASM 目标编译
- [ ] LSP 连接成功显示诊断
- [ ] Dashboard 实时显示编译进度

---

### T-04-14: bt-api — 认证中间件完善

- **ID**: `T-04-14`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P2
- **Sprint**: S-04-4

**描述**：完善 JWT 多租户模式和权限校验。

**产出文件**：
- `crates/bt-api/src/middleware/auth.rs` — 完善
- `crates/bt-api/src/middleware/project_context.rs`

**设计规格**（REF）：
- `05-Unified-API.md §2.10.4-2.10.7`

**前置依赖**（DEP）：
- `T-03-10`

**验收标准**（TST）：
- [ ] JWT RS256/HS256 验证
- [ ] ProjectContext 注入
- [ ] 权限矩阵强制执行
- [ ] E3003-E3006 错误码

---

### T-04-15: Phase 4 回归测试

- **ID**: `T-04-15`
- **状态**: TODO
- **估计**: 1 天
- **优先级**: P1
- **Sprint**: S-04-4

**验收标准**（TST）：
- [ ] `cargo build/test/fmt/clippy --workspace` 通过
- [ ] AI + 三后端 + LSP + 可观测性全部可用

---

## Phase 4 可选扩展 — 详细 Task 分解

### T-04-E01: bt-watch — FileWatcher + 文件变更检测

- **ID**: `T-04-E01`
- **来源**: F05 持续编译
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选（Phase 4 完成后追加）

**描述**：实现文件系统监控服务和变更检测。

**产出文件**：
- `crates/bt-watch/src/{lib.rs, watcher.rs, debounce.rs}`

**前置依赖**（DEP）：
- `T-04-06`：bt-api WebSocket 端点

**验收标准**（TST）：
- [ ] `WatchService::new()` + 文件系统监控（基于 notify crate）
- [ ] 跨平台：inotify/FSEvents/ReadDirectoryChanges
- [ ] 防抖批处理（100ms 窗口合并多次变更）
- [ ] `bt watch` 启动持续编译服务
- [ ] 显示文件变更日志

---

### T-04-E02: bt-watch — IncrementalPipeline + 增量编译集成

- **ID**: `T-04-E02`
- **来源**: F05 持续编译
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：增量编译管线集成，每次变更只编译受影响的部分。

**产出文件**：
- `crates/bt-watch/src/pipeline.rs`
- `crates/bt-service/src/compile_service.rs` — 更新

**前置依赖**（DEP）：
- `T-04-E01`
- `T-01-07`：bt-query Salsa 引擎

**验收标准**（TST）：
- [ ] `IncrementalPipeline::on_file_changed()` 增量编译
- [ ] Salsa 依赖追踪：仅重编译受影响 crate
- [ ] 编译结果 WebSocket 实时推送
- [ ] 增量编译 < 2s（小项目）

---

### T-04-E03: bt-watch — 诊断实时推送 + 编译时间线

- **ID**: `T-04-E03`
- **来源**: F05 持续编译
- **估计**: 3 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现诊断实时推送和编译时间线可视化。

**产出文件**：
- `crates/bt-watch/src/timeline.rs`
- `crates/bt-api/src/handlers/watch_ws.rs` — 更新

**前置依赖**（DEP）：
- `T-04-E02`

**验收标准**（TST）：
- [ ] 诊断结果 WebSocket 实时推送到 LSP/Dashboard
- [ ] 编译时间线记录（每个文件的变更历史）
- [ ] Dashboard 显示编译时间线可视化
- [ ] 异常编译检测（编译时间突增告警）
- [ ] `bt build --watch` 编译 + 持续监控

---

### T-04-E04: CompileFeatures 提取 + CompileProfileDatabase

- **ID**: `T-04-E04`
- **来源**: F06 AI 自适应策略
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现编译特征提取和编译档案数据库。

**产出文件**：
- `crates/bt-ai/src/adaptive/{mod.rs, features.rs, database.rs}`

**前置依赖**（DEP）：
- `T-04-02`：AIOrchestrator

**验收标准**（TST）：
- [ ] `CompileFeatures` 提取（~50 维特征向量）
- [ ] `CompileProfileDatabase` 持久化
- [ ] 编译档案记录到 EventStore
- [ ] 可导出为 ML 训练数据（`bt adaptive profile --export`）

---

### T-04-E05: PassRuleEngine + AdaptivePassScheduler

- **ID**: `T-04-E05`
- **来源**: F06 AI 自适应策略
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现基于规则引擎的 Pass 调度器。

**产出文件**：
- `crates/bt-ai/src/adaptive/{scheduler.rs, rules.rs}`

**前置依赖**（DEP）：
- `T-04-E04`

**验收标准**（TST）：
- [ ] `PassRuleEngine`：20+ 内置调度规则
- [ ] `AdaptivePassScheduler::recommend()` 推荐 Pass 序列
- [ ] 规则：小函数跳过内联 / 大量循环优先向量化 / CI 跳过 AI
- [ ] `bt build --adaptive` 自适应模式
- [ ] 规则引擎零延迟（<1ms）

---

### T-04-E06: CarbonTracker — 碳足迹追踪

- **ID**: `T-04-E06`
- **来源**: F08 绿色编译
- **估计**: 3 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现编译碳足迹估算。

**产出文件**：
- `crates/bt-telemetry/src/carbon.rs`

**前置依赖**（DEP）：
- `T-04-05`：bt-telemetry 全链路追踪

**验收标准**（TST）：
- [ ] `CarbonTracker::estimate()` 估算 CO₂ 排放
- [ ] 排放模型：CPU 时间 × 功耗 × 碳强度
- [ ] 直观类比：开车公里数 / 充手机次数
- [ ] `bt build --carbon` 显示碳足迹

---

### T-04-E07: GreenScheduler — 绿色调度

- **ID**: `T-04-E07`
- **来源**: F08 绿色编译
- **估计**: 2 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现绿色调度（高碳强度时推迟非紧急编译）。

**产出文件**：
- `crates/bt-telemetry/src/green_scheduler.rs`

**前置依赖**（DEP）：
- `T-04-E06`

**验收标准**（TST）：
- [ ] `GreenScheduler::should_compile()` 碳强度判断
- [ ] ElectricityMap API 集成
- [ ] 增量编译节省的碳排放显示
- [ ] 会话统计：本周/本月累计碳足迹
