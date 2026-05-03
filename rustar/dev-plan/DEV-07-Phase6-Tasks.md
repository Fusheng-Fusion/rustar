# Phase 6+ — 前瞻扩展（可选，主线交付后按需追加）

> **目标**：Phase 0-5 主线交付后，BlockType Rust 全功能可用。以下扩展按需追加。
> **估计**：约 32 周（160 天）| 5 个扩展 | 22 个 Task | 工期取决于选择哪些扩展
> **详细方案**：见 `future-extensions/` 目录各 Fxx 文档

---

## 扩展总览

| 扩展 | 工期 | Task 数 | 前置依赖 | 方案 |
|------|------|---------|---------|------|
| F03 形式化验证 | 8 周 | 5 个 | bt-ir, bt-dialect-rust | [F03](./../future-extensions/F03-Formal-Verification.md) |
| F04 编译时 Fuzzing | 5 周 | 4 个 | bt-ir, bt-passes | [F04](./../future-extensions/F04-Compile-Time-Fuzzing.md) |
| F07 Compiler Marketplace | 5 周 | 3 个 | bt-plugin-host, bt-api | [F07](./../future-extensions/F07-Compiler-Marketplace.md) |
| F10 浏览器内编译器 | 8 周 | 5 个 | bt-core, bt-ir（无 rustc） | [F10](./../future-extensions/F10-Browser-Compiler.md) |
| F12 跨语言反射 | 6 周 | 5 个 | bt-ir, bt-rustc-bridge, bt-ts2rs | [F12](./../future-extensions/F12-Cross-Language-Reflection.md) |

---

## F03: 形式化验证集成（8 周）

### T-06-F01: bt-contract — ContractAnnotation + IR 集成

- **ID**: `T-06-F01`
- **来源**: F03 形式化验证
- **估计**: 5 天
- **优先级**: P4

**描述**：实现 IR 合约标注类型和 IRModule 集成。

**产出文件**：
- `crates/bt-contract/src/{lib.rs, annotation.rs}`
- `crates/bt-ir/src/module.rs` — IRFunction 增加 contract 字段

**前置依赖**（DEP）：
- `T-01-01`：完整 IRType

**验收标准**（TST）：
- [ ] `ContractAnnotation` 数据结构（pre/postcondition/invariant）
- [ ] `ContractExpr` 表达式（比较/逻辑/量化/特殊）
- [ ] IRFunction 可携带 `Option<ContractAnnotation>`
- [ ] 序列化/反序列化

---

### T-06-F02: bt_rust Dialect — 合约操作码

- **ID**: `T-06-F02`
- **来源**: F03 形式化验证
- **估计**: 3 天
- **优先级**: P4

**描述**：在 bt_rust Dialect 中增加合约相关操作码。

**产出文件**：
- `crates/bt-dialect-rust/src/contract_ops.rs`

**前置依赖**（DEP）：
- `T-06-F01`
- `T-01-04`：bt_rust Dialect

**验收标准**（TST）：
- [ ] `contract_assert` 操作码（编译期检查）
- [ ] `contract_assume` 操作码（验证器提示）

---

### T-06-F03: SMT 求解器集成 + ContractVerifierPass

- **ID**: `T-06-F03`
- **来源**: F03 形式化验证
- **估计**: 5 天
- **优先级**: P4

**描述**：实现 SMT 求解器 trait 和 Z3 集成，以及合约验证 Pass。

**产出文件**：
- `crates/bt-contract/src/{solver.rs, verifier_pass.rs, z3_solver.rs}`

**前置依赖**（DEP）：
- `T-06-F02`

**验收标准**（TST）：
- [ ] `SMTSolver` trait（check_satisfiability/get_model）
- [ ] Z3 求解器实现
- [ ] `ContractVerifierPass` 在 BTIR 上验证合约
- [ ] 超时控制 + 最大路径数限制
- [ ] `bt verify` CLI 命令

---

### T-06-F04: SymbolicExecutor — 符号执行引擎

- **ID**: `T-06-F04`
- **来源**: F03 形式化验证
- **估计**: 5 天
- **优先级**: P4

**描述**：实现 BTIR 的符号执行引擎。

**产出文件**：
- `crates/bt-contract/src/symbolic_executor.rs`

**前置依赖**（DEP）：
- `T-06-F03`

**验收标准**（TST）：
- [ ] `SymbolicExecutor::execute()` 符号执行
- [ ] 路径爆炸控制（max_paths + max_depth）
- [ ] 反例生成（合约违反时的 counterexample）
- [ ] 验证覆盖度统计

---

### T-06-F05: Kani/Prusti 桥接 + 端到端测试

- **ID**: `T-06-F05`
- **来源**: F03 形式化验证
- **估计**: 2 天
- **优先级**: P4

**描述**：对接 Kani Rust Verifier 和 Prusti 的输出格式。

**产出文件**：
- `crates/bt-contract/src/verifier_bridge.rs`

**前置依赖**（DEP）：
- `T-06-F04`

**验收标准**（TST）：
- [ ] Kani 输出解析 → CompilerEvent
- [ ] Prusti 合约格式兼容
- [ ] `bt verify --coverage` 验证覆盖度
- [ ] 端到端测试通过

---

## F04: 编译时 Fuzzing（5 周）

### T-06-F06: bt-fuzz — IRMutator 语义保持变异

- **ID**: `T-06-F06`
- **来源**: F04 编译时 Fuzzing
- **估计**: 5 天
- **优先级**: P4

**描述**：实现 IR 语义保持变异器。

**产出文件**：
- `crates/bt-fuzz/src/{lib.rs, mutator.rs, strategies.rs}`

**前置依赖**（DEP）：
- `T-01-01`：完整 IRType
- `T-01-08`：Pass trait

**验收标准**（TST）：
- [ ] 8 种变异策略（常量替换/操作交换/分支翻转/死代码注入等）
- [ ] `IRMutator::mutate()` / `mutate_many()`
- [ ] 可配置变异策略权重
- [ ] `bt fuzz --mutants=1000` CLI 参数

---

### T-06-F07: bt-fuzz — DiffTestEngine 差异测试

- **ID**: `T-06-F07`
- **来源**: F04 编译时 Fuzzing
- **估计**: 5 天
- **优先级**: P4

**描述**：实现差异测试引擎，验证 Pass 等价性。

**产出文件**：
- `crates/bt-fuzz/src/{diff_engine.rs, equivalence.rs}`

**前置依赖**（DEP）：
- `T-06-F06`

**验收标准**（TST）：
- [ ] `DiffTestEngine::diff_test()` 双路径对比
- [ ] `PassEquivalenceTest` Pass 语义等价性验证
- [ ] 跨后端对比（LLVM vs Cranelift）
- [ ] `bt fuzz --pass=dce` 测试特定 Pass
- [ ] CI 模式：`bt fuzz --ci --mutants=100`

---

### T-06-F08: bt-fuzz — FuzzHarnessGenerator

- **ID**: `T-06-F08`
- **来源**: F04 编译时 Fuzzing
- **估计**: 3 天
- **优先级**: P4

**描述**：从 IRModule 自动提取 fuzz harness。

**产出文件**：
- `crates/bt-fuzz/src/{harness_gen.rs, interface.rs}`

**前置依赖**（DEP）：
- `T-06-F07`

**验收标准**（TST）：
- [ ] 从函数签名提取 FuzzableInterface
- [ ] 生成 libFuzzer 兼容的 harness
- [ ] `bt fuzz --harness` CLI 命令

---

### T-06-F09: bt-fuzz — 变异测试覆盖率

- **ID**: `T-06-F09`
- **来源**: F04 编译时 Fuzzing
- **估计**: 2 天
- **优先级**: P4

**描述**：实现变异测试覆盖率统计。

**产出文件**：
- `crates/bt-fuzz/src/coverage.rs`

**前置依赖**（DEP）：
- `T-06-F08`

**验收标准**（TST）：
- [ ] `MutationCoverageReport`（total/killed/survived/score）
- [ ] HTML 覆盖率报告生成
- [ ] `bt fuzz --mutation-score` CLI 命令

---

## F07: Compiler Marketplace（5 周）

### T-06-F10: bt-plugin-index — .btpkg 格式 + 本地注册表

- **ID**: `T-06-F10`
- **来源**: F07 Compiler Marketplace
- **估计**: 5 天
- **优先级**: P4

**描述**：定义插件包格式和本地注册表。

**产出文件**：
- `crates/bt-plugin-index/src/{lib.rs, package.rs, local_registry.rs}`

**前置依赖**（DEP）：
- `T-05-05`：bt-plugin-host WASM 插件

**验收标准**（TST）：
- [ ] `PluginPackage` 数据结构 + .btpkg 格式
- [ ] `PluginKind`：Dialect/Pass/LintRule/AIProvider
- [ ] 本地注册表：安装/卸载/列表
- [ ] `bt plugin install/list/uninstall` CLI 命令
- [ ] WASM 哈希验证 + 加载到沙箱

---

### T-06-F11: bt-plugin-index — 远程注册表客户端

- **ID**: `T-06-F11`
- **来源**: F07 Compiler Marketplace
- **估计**: 5 天
- **优先级**: P4

**描述**：实现远程注册表客户端（搜索/更新/版本管理）。

**产出文件**：
- `crates/bt-plugin-index/src/{registry_client.rs, version.rs}`

**前置依赖**（DEP）：
- `T-06-F10`

**验收标准**（TST）：
- [ ] `RegistryClient::search()` 搜索插件
- [ ] `RegistryClient::install()` 下载+验证+注册
- [ ] 版本管理（semver + 依赖解析）
- [ ] `blocktype-plugins.toml` 项目级配置
- [ ] `bt plugin update/search` CLI 命令

---

### T-06-F12: OpenAPI 自动生成 + Compiler-as-a-Service

- **ID**: `T-06-F12`
- **来源**: F07 Compiler Marketplace
- **估计**: 5 天
- **优先级**: P4

**描述**：自动生成 OpenAPI 规范和注册表客户端配置。

**产出文件**：
- `crates/bt-api/src/openapi.rs`
- `crates/bt-plugin-index/src/signature.rs`

**前置依赖**（DEP）：
- `T-06-F11`
- `T-00-07`：bt-api

**验收标准**（TST）：
- [ ] 从 axum router 自动生成 OpenAPI 规范
- [ ] 插件签名验证（sigstore）
- [ ] `bt server --public` 公开编译服务
- [ ] `bt plugin publish` 发布到注册表

---

## F10: 浏览器内编译器（8 周）

### T-06-F13: bt-ir-wasm — WASM 编译基础

- **ID**: `T-06-F13`
- **来源**: F10 浏览器内编译器
- **估计**: 5 天
- **优先级**: P4

**描述**：将 bt-core + bt-ir 编译为 WASM。

**产出文件**：
- `crates/bt-ir-wasm/src/{lib.rs, engine.rs}`
- `crates/bt-ir-wasm/Cargo.toml`

**前置依赖**（DEP）：
- `T-00-02`：bt-core
- `T-01-01`：bt-ir 完整 IRType

**验收标准**（TST）：
- [ ] bt-core + bt-ir 可编译为 WASM（wasm-pack build）
- [ ] `BtIrEngine` WASM 绑定（wasm_bindgen）
- [ ] 加载 IR 模块（`load_ir_module(json)`）
- [ ] 导出 IR 为 JSON（`export_ir_json(id)`）
- [ ] WASM 包体积优化（tree shaking + 压缩）

---

### T-06-F14: bt-ir-wasm — 浏览器端 IR 验证

- **ID**: `T-06-F14`
- **来源**: F10 浏览器内编译器
- **估计**: 5 天
- **优先级**: P4

**描述**：将 bt-ir-verifier 编译为 WASM。

**产出文件**：
- `crates/bt-ir-wasm/src/verifier.rs`

**前置依赖**（DEP）：
- `T-06-F13`
- `T-01-06`：bt-ir-verifier

**验收标准**（TST）：
- [ ] bt-ir-verifier 可编译为 WASM
- [ ] `BtIrEngine::verify_ir(id)` 浏览器端验证
- [ ] `cargo test -p bt-ir-wasm` 通过

---

### T-06-F15: bt-ir-wasm — 浏览器端 Pass + Dialect 降级

- **ID**: `T-06-F15`
- **来源**: F10 浏览器内编译器
- **估计**: 5 天
- **优先级**: P4

**描述**：将部分 Pass 和 Dialect 降级逻辑编译为 WASM。

**产出文件**：
- `crates/bt-ir-wasm/src/{pass_runner.rs, dialect_lower.rs}`

**前置依赖**（DEP）：
- `T-06-F14`

**验收标准**（TST）：
- [ ] Dialect 降级 Pass 可在 WASM 中运行
- [ ] `BtIrEngine::lower_dialects(id)` 浏览器端降级
- [ ] 浏览器端 IR 差异对比（`diff_ir(old, new)`）

---

### T-06-F16: bt-ir-wasm — 浏览器端 AI 规则引擎

- **ID**: `T-06-F16`
- **来源**: F10 浏览器内编译器
- **估计**: 3 天
- **优先级**: P4

**描述**：将 AI 规则引擎编译为 WASM（不含 LLM 部分）。

**产出文件**：
- `crates/bt-ir-wasm/src/ai_engine.rs`

**前置依赖**（DEP）：
- `T-06-F15`

**验收标准**（TST）：
- [ ] `RuleBasedProvider` 规则引擎 WASM 化
- [ ] `BtIrEngine::ai_analyze(id)` 浏览器端分析
- [ ] 规则引擎离线可用（不依赖网络）

---

### T-06-F17: TypeScript 绑定 + NPM 包发布

- **ID**: `T-06-F17`
- **来源**: F10 浏览器内编译器
- **估计**: 2 天
- **优先级**: P4

**描述**：生成 TypeScript 类型绑定并发布 NPM 包。

**产出文件**：
- `crates/bt-ir-wasm/pkg/`
- `bt-ir-wasm/package.json`

**前置依赖**（DEP）：
- `T-06-F16`

**验收标准**（TST）：
- [ ] TypeScript 声明文件（`.d.ts`）生成
- [ ] `@blocktype/ir-wasm` NPM 包
- [ ] 浏览器示例可运行

---

## F12: 跨语言编译时反射（6 周）

### T-06-F18: bt-reflection — ReflectionMetadata 格式

- **ID**: `T-06-F18`
- **来源**: F12 跨语言编译时反射
- **估计**: 5 天
- **优先级**: P4

**描述**：定义编译时反射元数据标准格式。

**产出文件**：
- `crates/bt-reflection/src/{lib.rs, metadata.rs}`

**前置依赖**（DEP）：
- `T-01-01`：IRType
- `T-01-04`：bt_rust Dialect

**验收标准**（TST）：
- [ ] `ReflectionMetadata` 数据结构
- [ ] `TypeReflection`：name/kind/fields/methods/generic_params
- [ ] `FunctionReflection`：params/return_type/async
- [ ] 序列化/反序列化

---

### T-06-F19: bt-reflection — IRModule 反射集成

- **ID**: `T-06-F19`
- **来源**: F12 跨语言编译时反射
- **估计**: 5 天
- **优先级**: P4

**描述**：将反射元数据集成到 IRModule 并从 Rust/TS 侧提取。

**产出文件**：
- `crates/bt-ir/src/module.rs` — IRModule 增加 reflection 字段
- `crates/bt-rustc-bridge/src/converter.rs` — 更新
- `crates/bt-ts2rs/src/annotation.rs` — 更新

**前置依赖**（DEP）：
- `T-06-F18`

**验收标准**（TST）：
- [ ] IRModule.reflection 可携带元数据
- [ ] MIR→BTIR 转换时自动提取反射信息
- [ ] bt-ts2rs 转译时提取 ts2rs 反射信息
- [ ] `bt reflect` CLI 命令

---

### T-06-F20: bt-reflection — SchemaGenerator 类型声明生成

- **ID**: `T-06-F20`
- **来源**: F12 跨语言编译时反射
- **估计**: 5 天
- **优先级**: P4

**描述**：从反射元数据生成跨语言类型声明。

**产出文件**：
- `crates/bt-reflection/src/{schema_gen.rs, type_mapper.rs}`

**前置依赖**（DEP）：
- `T-06-F19`

**验收标准**（TST）：
- [ ] 生成 TypeScript `.d.ts` 声明
- [ ] 生成 JSON Schema
- [ ] 内置类型映射：Rust ↔ TypeScript（50+ 映射）
- [ ] `bt reflect --ts-decls` CLI 命令

---

### T-06-F21: bt-reflection — 序列化/反序列化桥梁

- **ID**: `T-06-F21`
- **来源**: F12 跨语言编译时反射
- **估计**: 5 天
- **优先级**: P4

**描述**：从反射元数据自动生成跨语言序列化代码。

**产出文件**：
- `crates/bt-reflection/src/{serialization.rs, codegen.rs}`

**前置依赖**（DEP）：
- `T-06-F20`

**验收标准**（TST）：
- [ ] 支持 JSON/bincode/MessagePack 序列化
- [ ] 生成 Rust `Serialize`/`Deserialize` 代码
- [ ] 生成 TypeScript 序列化/反序列化代码
- [ ] 编译时绑定检查：Rust 字段变更 → TypeScript 告警

---

### T-06-F22: bt-reflection — 编译时反射 API

- **ID**: `T-06-F22`
- **来源**: F12 跨语言编译时反射
- **估计**: 3 天
- **优先级**: P4

**描述**：实现编译时可调用的类型查询 API。

**产出文件**：
- `crates/bt-reflection/src/query_api.rs`

**前置依赖**（DEP）：
- `T-06-F21`

**验收标准**（TST）：
- [ ] `bt_reflect::type_info::<T>()` 编译期查询
- [ ] `bt_reflect::fields::<T>()` 字段列表
- [ ] 跨语言类型查询：`cross_language_type_info("typescript", "User")`
- [ ] `bt reflect --query=User` CLI 命令