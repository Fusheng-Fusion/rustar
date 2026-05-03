# 04 — Rust Cargo Workspace 项目结构

## 4.1 Workspace 布局

```
blocktype-next/
├── Cargo.toml                         # workspace 根
├── rust-toolchain.toml                # 锁定 nightly 日期版本 + rustc-dev 组件（sysroot 依赖必需）
│
├── crates/
│   │
│   │  ─── Layer 1: 基础设施 ───
│   ├── bt-core/                       # 核心类型、诊断、SourceManager、错误码
│   ├── bt-service/                    # tower::Service 编排、CompileService
│   ├── bt-api/                        # RESTful API 层 (axum) + WebSocket + LSP 适配器
│   ├── bt-telemetry/                  # 可观测性 (OpenTelemetry)
│   ├── bt-event-store/                # Event Sourcing (事件持久化/回放)
│   ├── bt-query/                      # Salsa 风格增量查询引擎
│   │
│   │  ─── Layer 2: IR 层 ───
│   ├── bt-ir/                         # BTIR 核心 (类型/值/指令/模块/Builder)
│   ├── bt-ir-verifier/                # IR 验证
│   ├── bt-passes/                     # IR 优化 Pass + PassManager
│   │
│   │  ─── Layer 2: Dialect 层 ───
│   ├── bt-dialect-core/               # bt_core Dialect (内建)
│   ├── bt-dialect-rust/               # bt_rust Dialect (运行时注册)
│   ├── bt-dialect-ts/                 # bt_ts Dialect (运行时注册)
│   │
│   │  ─── Layer 3: Rust 前端对接（复用 rustc） ───
│   ├── bt-cargo/                      # Cargo 工作空间 + 依赖管理 + feature gate
│   ├── bt-rustc-bridge/               # rustc MIR → BTIR 桥接（核心适配层）
│   ├── bt-clippy-integration/         # Clippy 整合 + AI 增强（bt clippy）
│   ├── bt-proc-macro/                 # 过程宏加载/执行
│   ├── bt-std-bridge/                 # 标准库链接（rustup sysroot）
│   │
│   │  ─── Layer 2: TypeScript 前端（ts2rs 转译） ───
│   ├── bt-ts2rs/                      # TS→Rust 转译器（核心：复用官方 TS 编译器 API）
│   ├── blocktype-ts-runtime/          # TS 运行时兼容库（Rust 函数层，非 VM）
│   │
│   │  ─── Layer 2: 后端层 ───
│   ├── bt-backend-common/             # Backend trait + Registry（inkwell/cranelift 用）
│   ├── bt-backend-llvm/               # LLVM 后端 (inkwell)
│   ├── bt-backend-cranelift/          # Cranelift 后端
│   ├── bt-backend-rustc/              # rustc 原生 LLVM 后端（100% 兼容，不实现 Backend trait）
│   │
│   │  ─── Layer 2: AI 服务 ───
│   ├── bt-ai/                         # AI 编排器 + Provider + 预算 + 缓存
│   │
│   │  ─── Layer 2: 插件系统 ───
│   ├── bt-plugin-host/                # WASM 插件宿主 (沙箱隔离)
│   │
│   │  ─── CLI 入口 ───
│   └── bt-cli/                        # CLI (clap) + 服务器启动
│
├── ts2rs-rules/                       # 转译规则集（TypeScript AST → Rust AST 映射规则）
│   ├── rules/                         # 按语法类别分组的转译规则
│   └── tests/                         # 转译测试（源 TS → 期望 Rust 代码）
│
├── plugin-sdk/                        # 插件开发 SDK (WIT 定义)
│   ├── wit/
│   │   ├── compiler-pass.wit          # Pass 插件接口
│   │   └── ai-provider.wit            # AI Provider 插件接口
│   └── examples/                      # Rust/Go/C 插件示例
│
├── runtime/
│   └── rust-runtime/                  # Rust 运行时 stub
│
├── dashboard/                         # Web 监控仪表盘 (可选)
│   ├── package.json
│   └── src/
│
├── tests/
│   ├── integration/
│   └── snapshots/
│
└── docs/
    └── plan/
```

## 4.2 Crate 依赖关系

```
bt-cli
 ├── bt-cargo                         # Cargo 集成
 │    └── cargo_metadata + toml
 ├── bt-rustc-bridge                  # rustc 前端对接（MIR → BTIR，bt build 模式）
 │    ├── [sysroot] rustc_driver      # ← sysroot + extern crate（详见 08 §8.4）
 │    ├── [sysroot] rustc_interface
 │    ├── [sysroot] rustc_middle
 │    ├── [sysroot] rustc_mir
 │    ├── [sysroot] rustc_hir
 │    ├── [sysroot] rustc_session
 │    ├── [sysroot] rustc_span
 │    ├── [sysroot] rustc_errors
 │    ├── bt-ir
 │    ├── bt-dialect-rust
 │    └── bt-core
 ├── bt-clippy-integration            # Clippy 整合 + AI 增强（bt clippy 模式）★ 新增
 │    ├── clippy_lints                # Clippy 700+ lint（crates.io 正常依赖）
 │    ├── [sysroot] rustc_driver      # sysroot + extern crate
 │    ├── [sysroot] rustc_lint        # Lint 框架
 │    ├── [sysroot] rustc_interface
 │    ├── [sysroot] rustc_middle
 │    ├── bt-ai                       # AI 增强层
 │    ├── bt-event-store              # 事件记录
 │    └── bt-core
 ├── bt-proc-macro                    # 过程宏
 │    └── bt-core
 ├── bt-std-bridge                    # 标准库链接
 │    └── bt-core
 ├── bt-api
 │    ├── bt-service
 │    │    ├── bt-ir
 │    │    ├── bt-frontend-common
 │    │    ├── bt-backend-common
 │    │    ├── bt-dialect-core
 │    │    ├── bt-event-store
 │    │    ├── bt-query
 │    │    └── bt-core
 │    ├── bt-telemetry
 │    │    └── bt-core
 │    └── axum + tower + tokio (LSP 适配器内置于此 crate)
├── bt-ts2rs                          # TS→Rust 转译器
│    ├── Deno 进程（执行官方 TS 编译器）
│    ├── bt-ir                          # 引用 IRType（用于类型映射）
│    └── syn + quote                    # Rust AST 生成
├── blocktype-ts-runtime               # TS 运行时兼容库（Rust 函数层，非 VM）
│    └── bt-core
├── bt-backend-llvm
 │    ├── bt-backend-common
 │    │    └── bt-core
 │    ├── bt-ir
 │    └── inkwell
 ├── bt-backend-cranelift
 │    ├── bt-backend-common
 │    ├── bt-ir
 │    └── cranelift
├── bt-backend-rustc                   # rustc 原生 LLVM（不实现 Backend trait）
│    └── [sysroot] rustc_codegen_ssa   # rustc 内部 codegen
├── bt-passes
 ├── bt-dialect-rust
 │    ├── bt-ir
 │    └── bt-dialect-core
 ├── bt-dialect-ts
 │    ├── bt-ir
 │    └── bt-dialect-core
 ├── bt-ai
 │    └── bt-core
 ├── bt-plugin-host
 │    ├── bt-ir
 │    └── wasmtime + wit-bindgen
 ├── bt-event-store
 │    └── bt-core
 ├── bt-query
 │    ├── bt-ir
 │    └── bt-core
 └── clap
```

> **注意**：`bt-rustc-bridge` 是 BlockType 与 rustc 的唯一接口（bt build 模式），负责 MIR → BTIR 转换。
> `bt-clippy-integration` 是 Clippy 整合的唯一入口（bt clippy 模式），复用 Clippy 700+ 规则并叠加 AI 增强。
> Rust 前端全部复用 rustc，不再有自有 Rust 前端 crate。
> **TypeScript 前端通过 ts2rs 转译**：TS 源码→官方 TS 编译器（100% 兼容）→ ts2rs 转译器→ Rust 源码→
> bt-rustc-bridge 走完整 Rust 编译链路。详见本文件 §4.3。
>
> **Sysroot 依赖说明**：`[sysroot]` 标记的 rustc crate 通过 `#![feature(rustc_private)]` + `extern crate`
> 从 sysroot 引入，**不在 Cargo.toml 中声明**。`rust-toolchain.toml` 锁定 nightly 版本。
> Clippy lint 规则通过 `clippy_lints` crate（crates.io）正常依赖。
> 详见 [08-Rust-Ecosystem-Integration.md §8.4](./08-Rust-Ecosystem-Integration.md) 和 [§8.12](./08-Rust-Ecosystem-Integration.md)。

## 4.3 TypeScript 前端 — ts2rs 转译方案

> **核心思路**：不自建 TypeScript 编译器。通过 **ts2rs 转译**将 TypeScript 源码映射为等价的 Rust 源码，
> 然后复用 `bt-rustc-bridge` 走完整 Rust 编译链路。这与 Rust 前端"复用 rustc"的策略一致。

```
.ts source
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│                  bt-ts2rs 转译器                                │
│                                                              │
│  1. [Deno 进程] 调用官方 TypeScript 编译器 API                 │
│     ts.createProgram() + checker.getTypeAtLocation()          │
│     → 获取 100% 兼容的 AST + 完整类型信息                      │
│                                                              │
│  2. TypeScript AST → Rust AST（syn crate 的 File 类型）       │
│     ├── function → fn                                         │
│     ├── interface → struct + trait                            │
│     ├── class → struct + impl                                 │
│     ├── union type → enum                                     │
│     ├── async/await → async fn + await                        │
│     └── import/export → use/pub mod                           │
│                                                              │
│  3. 类型映射：number→f64, string→String, T|null→Option<T>     │
│  4. 注入 blocktype-ts-runtime 辅助函数                        │
│  5. rustfmt 格式化输出 .rs 文件                               │
└──────────────────────────────────────────────────────────────┘
    │
    ▼  生成的 .rs 源码
┌──────────────────────────────────────────────────────────────┐
│              完整 Rust 编译管线（100% 复用）                    │
│                                                              │
│  bt-rustc-bridge → MIR → BTIR → AI Pass → 优化 → 后端 → 链接 │
│                                                              │
│  自动获得：AI 分析 / Clippy / OpenTelemetry / EventStore      │
│  / 增量编译 / Salsa 查询 / 三后端 / WASM 输出等全部能力       │
└──────────────────────────────────────────────────────────────┘
```

> **关键设计点**：bt-ts2rs 转译器通过单进程 Deno 调用官方 `typescript` npm 包。
> 不需要进程池/负载均衡/心跳（Deno 仅作为 TS 编译器 API 宿主，非运行时）。
> 不可映射的 TS 模式（如 `eval()`、原型链继承）拒绝编译并提示替代写法。
> 完整设计参见 `02-Core-Types.md §7.4`（bt_ts Dialect 标注层）。

## 4.4 各 Crate 职责

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-core` | 诊断、SourceManager、TargetTriple、错误码体系 | ~1,800 |
| `bt-service` | tower::Service 编排、CompileService、中间件链 | ~1,500 |
| `bt-api` | axum HTTP 服务器、路由、handler、WebSocket、LSP 适配器 | ~3,500 |
| `bt-telemetry` | OpenTelemetry 集成、PipelineNode、指标 | ~1,500 |
| `bt-event-store` | Event Sourcing 事件持久化、回放、导出、订阅 | ~1,500 |
| `bt-query` | Salsa 风格增量查询引擎、依赖追踪、缓存 | ~2,500 |
| `bt-ir` | IR 类型/值/指令/模块/Builder/DialectInstruction | ~4,500 |
| `bt-ir-verifier` | IR 验证、类型检查、等价性 | ~1,000 |
| `bt-passes` | Pass trait (含依赖声明)、PassManager、优化 Pass | ~3,500 |
| `bt-dialect-core` | bt_core Dialect 定义 + Dialect trait | ~800 |
| `bt-dialect-rust` | bt_rust Dialect (运行时注册) + 降级规则 | ~1,500 |
| `bt-dialect-ts` | bt_ts Dialect（标注/分析层） + 转译标注规则 | ~1,200 |
| `bt-cargo` | Cargo 工作空间解析、依赖图、feature gate、构建计划 | ~2,000 |
| `bt-rustc-bridge` | rustc_driver 集成、MIR→BTIR 转换、类型映射 | ~4,000 |
| `bt-proc-macro` | 过程宏 .so 加载/执行、沙箱化 | ~1,500 |
| `bt-std-bridge` | rustup sysroot 检测、标准库链接 | ~500 |
| `bt-clippy-integration` | Clippy 700+ 规则整合 + AI 增强层 | ~1,500 |
| `bt-ts2rs` | TypeScript→Rust 转译器（复用官方 TS 编译器 API） | ~2,500 |
| `blocktype-ts-runtime` | TS 运行时兼容库（Rust 函数层：console/Math/JSON/Date/Array/String） | ~1,000 |
| `bt-backend-common` | Backend trait + Registry（inkwell/cranelift 用） | ~500 |
| `bt-backend-llvm` | inkwell 封装，x86_64/AArch64 | ~3,000 |
| `bt-backend-cranelift` | Cranelift 封装，WASM 目标 | ~2,000 |
| `bt-backend-rustc` | **rustc 原生 LLVM 后端**（委托 rustc codegen，不实现 Backend trait） | ~800 |
| `bt-ai` | AI 编排器 + 多 Provider + 预算 + 缓存 + 规则引擎 | ~3,000 |
| `bt-plugin-host` | WASM 插件宿主、沙箱隔离、WIT 接口 | ~1,500 |
| `bt-cli` | CLI 入口 (clap) | ~500 |
| **Rust 总计** | | **~48,600** |
| `ts2rs-rules/` | 转译规则集（TS AST → Rust AST 映射规则） | ~2,000 |
| `plugin-sdk/` | WIT 定义 + 插件示例 | ~500 |
| **项目总计** | | **~51,100** |
