# Phase 5 — 高级特性 + TypeScript 前端

> **目标**：BlockType Rust 全功能可用
> **估计**：~12 周（60 天）| 5 个 Sprint | 15 个 Task
> **里程碑**：优化 Pass + 插件系统 + 增量编译 + AI 流式 + TS 前端 + 端到端测试

---

## Sprint 5-1: bt-passes 优化 Pass + Dialect 降级完善（2 周）

### T-05-01: bt-passes — DCE (Dead Code Elimination)

- **ID**: `T-05-01`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-05-1

**描述**：实现死代码消除 Pass。

**产出文件**：
- `crates/bt-passes/src/passes/dce.rs`

**前置依赖**（DEP）：
- `T-01-08`：Pass trait + PassManager

**验收标准**（TST）：
- [ ] `DCEPass` impl `Pass`
- [ ] 移除不可达基本块
- [ ] 移除无副作用的无用指令
- [ ] 保留有副作用的指令（Store/Call）
- [ ] 测试：IR 中无用指令被正确消除

---

### T-05-02: bt-passes — 常量折叠 + 内联

- **ID**: `T-05-02`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P0
- **Sprint**: S-05-1

**描述**：实现常量折叠和函数内联 Pass。

**产出文件**：
- `crates/bt-passes/src/passes/const_fold.rs`
- `crates/bt-passes/src/passes/inline.rs`

**前置依赖**（DEP）：
- `T-01-08`

**验收标准**（TST）：
- [ ] `ConstFoldPass`: 编译期计算常量表达式
- [ ] `InlinePass`: 内联小函数（阈值可配置）
- [ ] 支持 OptLevel 控制激进度
- [ ] Pass 依赖声明正确

---

### T-05-03: bt-passes — CFG 简化

- **ID**: `T-05-03`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P1
- **Sprint**: S-05-1

**描述**：实现控制流图简化 Pass。

**产出文件**：
- `crates/bt-passes/src/passes/cfg_simplify.rs`

**前置依赖**（DEP）：
- `T-01-08`

**验收标准**（TST）：
- [ ] 合并只有一个前驱/后继的基本块
- [ ] 消除空基本块
- [ ] 简化分支条件（常量条件）

---

### T-05-04: Dialect 降级完善

- **ID**: `T-05-04`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-05-1

**描述**：bt_rust/bt_ts → bt_core 完整降级。

**产出文件**：
- `crates/bt-dialect-rust/src/lower.rs` — 完善
- `crates/bt-dialect-ts/src/lower.rs` — 完善

**前置依赖**（DEP）：
- `T-01-04`, `T-01-05`

**验收标准**（TST）：
- [ ] bt_rust 10 个操作码全部可降级
- [ ] bt_ts 10 个操作码全部可降级
- [ ] 降级后的 IR 通过验证
- [ ] 降级后指令数减少（统计）

---

## Sprint 5-2: 插件系统 + 增量编译（2 周）

### T-05-05: bt-plugin-host — WASM 插件宿主

- **ID**: `T-05-05`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-05-2

**描述**：实现 WASM Component Model 插件宿主。

**产出文件**：
- `crates/bt-plugin-host/src/{lib.rs, host.rs, sandbox.rs, instance.rs}`
- `plugin-sdk/wit/compiler-pass.wit`
- `plugin-sdk/wit/ai-provider.wit`

**设计规格**（REF）：
- `04-Project-Structure.md §4.1` — plugin-sdk
- `01-Design-Philosophy.md §1.1 P3` — 可插拔原则

**前置依赖**（DEP）：
- `T-01-08`：Pass trait

**验收标准**（TST）：
- [ ] `PluginHost::load(path)` 加载 WASM 插件
- [ ] WIT 接口：compiler-pass / ai-provider
- [ ] 沙箱隔离：内存/CPU 限制
- [ ] 插件可注册为 Pass
- [ ] `cargo test -p bt-plugin-host` 通过

---

### T-05-06: plugin-sdk — WIT 定义 + 插件示例

- **ID**: `T-05-06`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P2
- **Sprint**: S-05-2

**描述**：WIT 接口定义和插件开发示例。

**产出文件**：
- `plugin-sdk/wit/*.wit`
- `plugin-sdk/examples/hello_pass.rs`

**前置依赖**（DEP）：
- `T-05-05`

**验收标准**（TST）：
- [ ] compiler-pass.wit 定义完整
- [ ] ai-provider.wit 定义完整
- [ ] Rust 插件示例可编译和加载

---

### T-05-07: Salsa 增量编译完善

- **ID**: `T-05-07`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-05-2

**描述**：完善 Salsa 增量编译，实现文件修改后秒级重编译。

**产出文件**：
- `crates/bt-query/src/engine.rs` — 完善
- `crates/bt-cargo/src/incremental.rs` — 完善

**前置依赖**（DEP）：
- `T-01-07`, `T-02-04`

**验收标准**（TST）：
- [ ] 文件修改后增量重编译 < 2s（小项目）
- [ ] Salsa 依赖追踪精确到函数级
- [ ] 增量编译结果与全量编译一致
- [ ] 依赖图可视化

---

### T-05-08: AI 流式分析 + 自动变换

- **ID**: `T-05-08`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-05-2

**描述**：完善 AI 流式分析和 AutoApply 自动变换。

**产出文件**：
- `crates/bt-ai/src/stream.rs`
- `crates/bt-ai/src/auto_apply.rs`

**前置依赖**（DEP）：
- `T-04-04`

**验收标准**（TST）：
- [ ] `analyze_stream()` 逐 chunk 输出
- [ ] `AutoApply` 自动应用高置信度（>0.9）变换
- [ ] 变换后 IR 通过验证
- [ ] 变换记录到 EventStore

---

## Sprint 5-3: TypeScript 前端 — ts2rs 核心转译器（2 周）

### T-05-09: bt-ts2rs — 官方 TS 编译器桥接

- **ID**: `T-05-09`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-05-3

**描述**：实现 bt-ts2rs 的 Deno 进程封装，调用官方 `typescript` npm 包的编译器 API 获取类型信息。

**产出文件**：
- `crates/bt-ts2rs/src/lib.rs` — bt-ts2rs 入口
- `crates/bt-ts2rs/src/ts_host.rs` — Deno 进程管理 + JSON-RPC 通信
- `crates/bt-ts2rs/ts-host/main.ts` — Deno 侧 TS 编译器 API 封装脚本

**前置依赖**（DEP）：
- `T-00-02`：bt-core

**验收标准**（TST）：
- [ ] `TsHost::transpile(source, fileName)` 通过 Deno 子进程调用 `ts.createProgram()`
- [ ] 返回 `TranspileResult`（类型信息 + AST 摘要 + 诊断）
- [ ] 100% TypeScript 语法兼容（基于官方编译器）
- [ ] 错误诊断透传：TS 类型错误原样返回
- [ ] docker/npm 环境无关（通过 `npm:typescript` 包引用）
- [ ] `cargo test -p bt-ts2rs` 通过

---

### T-05-10: bt-ts2rs — TS AST → Rust AST 映射

- **ID**: `T-05-10`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-05-3

**描述**：实现 TypeScript AST 到 Rust AST（syn crate）的核心映射逻辑。

**产出文件**：
- `crates/bt-ts2rs/src/ast_mapper.rs` — AST 映射器
- `crates/bt-ts2rs/src/type_mapper.rs` — 类型映射器
- `ts2rs-rules/rules/` — 转译规则定义

**前置依赖**（DEP）：
- `T-05-09`

**设计规格**（REF）：
- `04-Project-Structure.md §4.3` — ts2rs 架构

**验收标准**（TST）：
- [ ] 函数声明映射：`function add(a: number): number` → `fn add(a: f64) -> f64`
- [ ] 接口映射：`interface User { name: string }` → `struct User { name: String }`
- [ ] 联合类型映射：`T | null` → `Option<T>`
- [ ] 泛型映射：`function id<T>(x: T): T` → `fn id<T>(x: T) -> T`
- [ ] async/await 映射
- [ ] import/export 映射为 use/pub mod
- [ ] 生成的 Rust 代码通过 `syn::parse_file()` 验证和 `rustfmt`
- [ ] 测试：输入 TS → 输出 Rust → 编译通过（用 rustc check）

---

## Sprint 5-4: TypeScript 前端 — 类型映射 + 运行时兼容（2 周）

### T-05-11: bt-ts2rs — 完整类型映射 + 模式覆盖

- **ID**: `T-05-11`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P1
- **Sprint**: S-05-4

**描述**：覆盖更复杂的 TypeScript 模式到 Rust 的映射。

**产出文件**：
- `crates/bt-ts2rs/src/mappers/class.rs` — class 映射
- `crates/bt-ts2rs/src/mappers/enum.rs` — enum 映射
- `crates/bt-ts2rs/src/mappers/control_flow.rs` — 控制流映射
- `crates/bt-ts2rs/src/mappers/pattern.rs` — 模式匹配映射

**前置依赖**（DEP）：
- `T-05-10`

**验收标准**（TST）：
- [ ] `class` 映射为 `struct` + `impl`（方法展平）
- [ ] `class extends` 映射为 trait 组合
- [ ] TypeScript `enum` 映射为 Rust `enum`
- [ ] `try/catch` 映射为 `Result<T, E>` + `match`
- [ ] `for...of` 映射为 `for x in iter`
- [ ] `?.` 可选链映射为 `if let Some(x) = a?.b()`
- [ ] `T | U` 联合类型映射为 Rust enum
- [ ] 不可映射模式（eval/any/原型链继承）拒绝编译并提示替代写法
- [ ] 100+ 转译测试用例（含边缘情况）

---

### T-05-12: blocktype-ts-runtime — TypeScript 运行时兼容库

- **ID**: `T-05-12`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P2
- **Sprint**: S-05-4

**描述**：实现 TypeScript 运行时兼容库（Rust 函数层封装，非 JS VM）。

**产出文件**：
- `crates/blocktype-ts-runtime/src/{lib.rs, console.rs, math.rs, json.rs, date.rs, array_ext.rs, string_ext.rs}`

**设计规格**（REF）：
- `02-Core-Types.md §7.4` — bt_ts Dialect 标注层

**验收标准**（TST）：
- [ ] `console.log(msg)` → `println!("{}", msg)`
- [ ] `Math.floor/random/min/max` → `f64` 原生方法封装
- [ ] `JSON.parse/stringify` → `serde_json`
- [ ] `Date.now()` → `std::time::SystemTime`
- [ ] `Array.push/pop/map/filter/reduce` → `Vec` 扩展 trait
- [ ] `String.split/trim/startsWith/slice` → `str` 扩展 trait
- [ ] 所有函数可选 tracing 支持（可观测性）
- [ ] `cargo test -p blocktype-ts-runtime` 通过

---

### T-05-13: bt-ts2rs — 集成 bt_ts 标注 + CLI 集成

- **ID**: `T-05-13`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-05-4

**描述**：bt-ts2rs 集成到 bt-cli + bt_ts Dialect 标注生成。

**产出文件**：
- `crates/bt-ts2rs/src/bridge.rs` — ts2rs Frontend trait 实现
- `crates/bt-ts2rs/src/annotation.rs` — bt_ts 标注生成

**前置依赖**（DEP）：
- `T-05-11`

**验收标准**（TST）：
- [ ] `bt-ts2rs` impl `Frontend` trait
- [ ] 转译过程中生成 `#[bt_ts_origin(...)]` 标注（bt_ts Dialect 标注层）
- [ ] 生成的 Rust 文件通过 `bt-rustc-bridge` 编译
- [ ] `bt build app.ts` 可编译 TypeScript 文件
- [ ] CLI 隐藏内部转译细节，用户无感知

---

## Sprint 5-5: 端到端集成 + 交付（2 周）

### T-05-14: 端到端集成测试 — Rust + TS 双前端

- **ID**: `T-05-14`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-05-5

**描述**：Rust + TypeScript（通过 ts2rs 转译）双前端端到端集成测试。

**产出文件**：
- `tests/integration/test_dual_frontend.rs`
- `tests/fixtures/ts_project/`

**前置依赖**（DEP）：
- `T-05-13`, `T-05-12`

**验收标准**（TST）：
- [ ] Rust 项目编译运行
- [ ] TypeScript 代码通过 ts2rs 转译 → bt-rustc-bridge 编译运行
- [ ] TS 转译 → 编译 → 运行结果与预期一致
- [ ] 双前端结果可对比
- [ ] API 驱动整个流程
- [ ] EventStore 记录完整事件链
- [ ] bt_ts 标注出现在生成的 IR 中（供 AI 分析层使用）

---

### T-05-15: Phase 5 回归测试 + 最终交付

- **ID**: `T-05-15`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P0
- **Sprint**: S-05-5

**验收标准**（TST）：
- [ ] `cargo build/test/fmt/clippy --workspace` 通过
- [ ] 全量集成测试通过（含 TS→Rust 转译 + Rust 编译）
- [ ] README.md 完整文档
- [ ] CONTRIBUTING.md 更新
- [ ] CI 配置完成
- [ ] BlockType Rust 全功能可用

---

## Phase 5 可选扩展 — 详细 Task 分解

> F03/F04/F07/F10/F12 的 Task 分解见 `DEV-07-Phase6-Tasks.md`。

### T-05-E01: bt-build-cache — Redis 后端实现

- **ID**: `T-05-E01`
- **来源**: F01 分布式编译缓存
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选（Phase 5 完成后追加）

**描述**：实现 Redis 缓存后端，支持团队共享编译缓存。

**产出文件**：
- `crates/bt-build-cache/src/redis_backend.rs`

**前置依赖**（DEP）：
- `T-02-E01`：本地缓存层

**验收标准**（TST）：
- [ ] `RedisBackend` impl `CacheBackend`
- [ ] Redis 连接池管理
- [ ] 缓存条目 TTL 管理
- [ ] 序列化/反序列化（bincode）
- [ ] `bt build --cache=redis` CLI 参数

---

### T-05-E02: bt-build-cache — S3 后端实现

- **ID**: `T-05-E02`
- **来源**: F01 分布式编译缓存
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现 S3 缓存后端，支持 CI 远程缓存。

**产出文件**：
- `crates/bt-build-cache/src/s3_backend.rs`

**前置依赖**（DEP）：
- `T-05-E01`

**验收标准**（TST）：
- [ ] `S3Backend` impl `CacheBackend`
- [ ] S3 兼容存储（AWS/MinIO）
- [ ] 批量读写 + 异步上传
- [ ] `bt build --cache=s3` CLI 参数

---

### T-05-E03: bt-build-cache — 缓存统计 + EventStore 记录

- **ID**: `T-05-E03`
- **来源**: F01 分布式编译缓存
- **估计**: 3 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现缓存统计和 Dashboard 可视化。

**产出文件**：
- `crates/bt-build-cache/src/stats.rs`
- `crates/bt-event-store/src/events.rs` — 更新

**前置依赖**（DEP）：
- `T-05-E02`

**验收标准**（TST）：
- [ ] `CacheStats`：命中率/节省时间/节省带宽
- [ ] 缓存事件记录到 EventStore
- [ ] Dashboard 缓存命中/未命中图表
- [ ] `bt cache stats` CLI 命令

---

### T-05-E04: BackendSelector — 自适应后端选择

- **ID**: `T-05-E04`
- **来源**: F06 AI 自适应策略
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现基于代码特征的自适应后端选择。

**产出文件**：
- `crates/bt-ai/src/adaptive/backend_selector.rs`

**前置依赖**（DEP）：
- `T-04-E05`：规则引擎版调度器
- `T-04-09`：bt-backend-rustc

**验收标准**（TST）：
- [ ] `BackendSelector::recommend_backend()` 推荐后端
- [ ] 规则：小项目→Cranelift，大项目→LLVM
- [ ] 混合后端模式：热函数 LLVM，冷函数 Cranelift
- [ ] `bt build --backend=auto` 完整工作流

---

### T-05-E05: ONNX ML 模型推理引擎

- **ID**: `T-05-E05`
- **来源**: F06 AI 自适应策略
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：集成 ONNX ML 模型推理引擎，实现基于历史编译数据的 Pass 调度学习。

**产出文件**：
- `crates/bt-ai/src/adaptive/{model.rs, onnx_engine.rs}`

**前置依赖**（DEP）：
- `T-05-E04`

**验收标准**（TST）：
- [ ] `PassModelEngine` trait（ONNX/tract/ort）
- [ ] 特征向量→模型推理 pipeline
- [ ] 模型推理 < 1ms
- [ ] 历史数据导出为训练集

---

### T-05-E06: CompileTimeQuery — 编译时 AI 查询

- **ID**: `T-05-E06`
- **来源**: F11 编译器 LLM 协作
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现编译时"暂停-询问-继续"的 AI 查询机制。

**产出文件**：
- `crates/bt-ai/src/collaboration/{mod.rs, query.rs}`

**前置依赖**（DEP）：
- `T-04-02`：AIOrchestrator

**验收标准**（TST）：
- [ ] `CompileTimeQuery` 数据结构 + LLM 查询
- [ ] `AmbiguityDetector` 歧义检测（规则引擎 + 不确定性阈值）
- [ ] 非阻塞查询：LLM 查询期间编译器继续其他任务
- [ ] 超时机制（默认 5s 超时后使用默认策略）
- [ ] `bt build --ai=collaborate` CLI 参数

---

### T-05-E07: InteractiveExplain — 对话式错误解释

- **ID**: `T-05-E07`
- **来源**: F11 编译器 LLM 协作
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现基于编译上下文的对话式错误解释。

**产出文件**：
- `crates/bt-ai/src/explain/{mod.rs, chat.rs}`

**前置依赖**（DEP）：
- `T-05-E06`

**验收标准**（TST）：
- [ ] `InteractiveExplain::chat()` 对话接口
- [ ] 自动注入编译上下文（前端/目标/诊断/管线状态）
- [ ] `bt explain --interactive` 对话模式
- [ ] 支持追问、代码示例生成、修复建议
- [ ] 所有交互记录到 EventStore（可审计）

---

### T-05-E08: LLM 查询审计 + 预算管理

- **ID**: `T-05-E08`
- **来源**: F11 编译器 LLM 协作
- **估计**: 3 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现 LLM 查询的预算管理和审计。

**产出文件**：
- `crates/bt-ai/src/collaboration/{budget.rs, audit.rs}`

**前置依赖**（DEP）：
- `T-05-E07`

**验收标准**（TST）：
- [ ] 每次编译最大 LLM 查询次数限制
- [ ] 高置信度自动应用（>0.95）
- [ ] 低置信度等待用户确认
- [ ] 审计日志：所有 LLM 交互可回放
- [ ] Dashboard 显示 AI 查询统计
