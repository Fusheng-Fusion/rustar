# Phase 3 — Proc-Macro + 生态兼容 + Clippy 整合

> **目标**：主流 Rust 项目编译通过 + Clippy 700+ 规则整合 + AI 增强层
> **估计**：~9 周（46 天）| 4 个 Sprint | 14 个 Task
> **里程碑**：serde/tokio/axum 项目编译通过；`bt clippy` 输出与 `cargo clippy` 一致 + AI 增强

---

## Sprint 3-1: bt-proc-macro 基础（2 周）

### T-03-01: bt-proc-macro — .so 加载 + dlopen + 执行

- **ID**: `T-03-01`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-03-1

**描述**：实现过程宏 .so 动态库加载和执行。

**产出文件**：
- `crates/bt-proc-macro/src/{lib.rs, host.rs, loader.rs}`

**设计规格**（REF）：
- `10-Phased-Roadmap.md §Phase 3`

**前置依赖**（DEP）：
- `T-00-02`：bt-core

**设计规格**（REF）：
- `08-Rust-Ecosystem-Integration.md §8.3.3` — bt-proc-macro 设计

**验收标准**（TST）：
- [ ] `ProcMacroHost::new()` 构造
- [ ] `load(path: &Path)` 通过 dlopen 加载 .so
- [ ] `execute_derive(name, input)` 执行 derive macro
- [ ] `execute_attribute(name, input)` 执行 attribute macro
- [ ] `execute_func_like(name, input)` 执行 function-like macro
- [ ] 自定义 derive macro 编译运行
- [ ] `cargo test -p bt-proc-macro` 通过

---

### T-03-02: bt-proc-macro — 常用 proc-macro 透传

- **ID**: `T-03-02`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P0
- **Sprint**: S-03-1

**描述**：确保常用 proc-macro（serde_derive 等）可透传执行。

**产出文件**：
- `crates/bt-proc-macro/src/compat.rs`
- `tests/fixtures/serde_project/`

**前置依赖**（DEP）：
- `T-03-01`

**验收标准**（TST）：
- [ ] `#[derive(Serialize, Deserialize)]` 编译通过
- [ ] `#[derive(Debug, Clone)]` 编译通过
- [ ] `#[tokio::main]` 编译通过
- [ ] 使用 serde 的项目编译并正确序列化/反序列化

---

### T-03-03: bt-proc-macro — 沙箱化（子进程执行）

- **ID**: `T-03-03`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-03-1

**描述**：proc-macro 在子进程中执行，panic 不崩溃编译器。

**产出文件**：
- `crates/bt-proc-macro/src/sandbox.rs`

**前置依赖**（DEP）：
- `T-03-01`

**验收标准**（TST）：
- [ ] proc-macro 在子进程执行
- [ ] panic 被捕获，不崩溃主进程
- [ ] 超时限制（默认 60s）
- [ ] 内存限制（默认 512MB）
- [ ] 测试：恶意 proc-macro（panic/无限循环）被安全处理

---

## Sprint 3-2: 生态兼容（2 周）

### T-03-04: top-100 crate 兼容性测试

- **ID**: `T-03-04`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-03-2

**描述**：测试 top-100 crate 编译兼容性。

**产出文件**：
- `tests/compat/compat_test.rs`
- `tests/compat/report.md`

**前置依赖**（DEP）：
- `T-03-02`, `T-02-06`

**验收标准**（TST）：
- [ ] serde/tokio/axum/clap/anyhow/thiserror 项目编译通过
- [ ] 生成兼容性报告：通过/失败/部分通过
- [ ] 失败的 crate 有具体错误分析

---

### T-03-05: bt-rustc-bridge — proc-macro 集成到编译管线

- **ID**: `T-03-05`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-03-2

**描述**：将 bt-proc-macro 集成到 bt-rustc-bridge 的编译流程中。

**产出文件**：
- `crates/bt-rustc-bridge/src/bridge.rs` — 更新

**前置依赖**（DEP）：
- `T-03-01`, `T-00-11`

**验收标准**（TST）：
- [ ] 编译流程中自动检测和加载 proc-macro
- [ ] 宏展开后继续正常编译流程

---

## Sprint 3-3: Clippy 整合（2 周）

### T-03-06: bt-clippy-integration — Clippy lint 注册

- **ID**: `T-03-06`
- **状态**: TODO
- **估计**: 5 天
- **优先级**: P0
- **Sprint**: S-03-3

**描述**：实现 bt-clippy-integration crate，整合 Clippy 700+ lint 规则。

**产出文件**：
- `crates/bt-clippy-integration/src/{lib.rs, callbacks.rs, registry.rs}`

**设计规格**（REF）：
- `08-Rust-Ecosystem-Integration.md §8.12` — Clippy 整合方案
- `appendix/A3-BlockType-vs-Clippy-Analysis.md`

**前置依赖**（DEP）：
- `T-00-09`：bt-rustc-bridge 骨架

**验收标准**（TST）：
- [ ] `BtClippyCallbacks` impl `rustc_driver::Callbacks`
- [ ] 注册 Clippy 700+ lint passes
- [ ] `bt clippy` 输出与 `cargo clippy` 一致
- [ ] lint 结果记录到 EventStore
- [ ] `cargo test -p bt-clippy-integration` 通过

---

### T-03-07: bt-clippy-integration — BtClippyEnhancer AI 增强层

- **ID**: `T-03-07`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P1
- **Sprint**: S-03-3

**描述**：在 Clippy 结果上叠加 AI 解释和增强建议。

**产出文件**：
- `crates/bt-clippy-integration/src/enhancer.rs`
- `crates/bt-clippy-integration/src/rules.rs` — ~20 条补充规则

**设计规格**（REF）：
- `08-Rust-Ecosystem-Integration.md §8.12` — AI 增强
- `09-Frontend-Innovation-Extension.md §9.3.5`

**前置依赖**（DEP）：
- `T-03-06`

**验收标准**（TST）：
- [ ] `BtClippyEnhancer` 接收 Clippy 结果 + 叠加 AI 分析
- [ ] ~20 条补充规则（跨文件/架构级）
- [ ] AI 解释叠加输出（Clippy message + AI explanation）
- [ ] `bt clippy --ai=auto` 启用 AI 增强

---

### T-03-08: bt-cli — `bt clippy` 命令

- **ID**: `T-03-08`
- **状态**: TODO
- **估计**: 1 天
- **优先级**: P0
- **Sprint**: S-03-3

**描述**：添加 `bt clippy` CLI 命令。

**产出文件**：
- `crates/bt-cli/src/commands/clippy.rs`

**前置依赖**（DEP）：
- `T-03-06`, `T-00-14`

**验收标准**（TST）：
- [ ] `bt clippy` 运行 Clippy lint
- [ ] `bt clippy --ai=auto` 叠加 AI 增强

---

## Sprint 3-4: 生态验证 + 回归（2 周）

### T-03-09: Clippy 兼容性测试

- **ID**: `T-03-09`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-03-4

**描述**：验证 `bt clippy` 在真实项目上的输出一致性。

**验收标准**（TST）：
- [ ] `bt clippy` 与 `cargo clippy` 输出对比（同一项目）
- [ ] 至少 5 个真实项目对比通过
- [ ] AI 增强输出格式正确

---

### T-03-10: bt-api — AI 操作 API + 认证中间件骨架

- **ID**: `T-03-10`
- **状态**: TODO
- **估计**: 4 天
- **优先级**: P1
- **Sprint**: S-03-4

**描述**：添加 `/api/v1/ai/*` 端点和认证中间件骨架（仅 `none` + `api_key` 模式）。JWT 多租户模式在 T-04-13 中完善。

**产出文件**：
- `crates/bt-api/src/handlers/ai.rs`
- `crates/bt-api/src/middleware/auth.rs` — 认证中间件骨架（none + api_key）

**设计规格**（REF）：
- `05-Unified-API.md §2.5` — AI 操作 API
- `05-Unified-API.md §2.10.1-2.10.3` — 设计原则 + 认证模式 + API Key

**前置依赖**（DEP）：
- `T-00-07`

**验收标准**（TST）：
- [ ] `POST /api/v1/ai/analyze|optimize|explain|chat` 端点
- [ ] `GET /api/v1/ai/providers` + `GET /api/v1/ai/budget`
- [ ] SSE 流式响应支持
- [ ] 认证中间件：`none`（默认放行）+ `api_key`（Bearer token 校验）
- [ ] 权限矩阵基础校验（reader/compiler/admin）

---

### T-03-11: bt-backend-llvm — AArch64 目标支持

- **ID**: `T-03-11`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P1
- **Sprint**: S-03-4

**描述**：LLVM 后端添加 AArch64 目标支持。

**前置依赖**（DEP）：
- `T-02-06`

**验收标准**（TST）：
- [ ] `targets()` 返回含 aarch64
- [ ] AArch64 目标代码生成正确

---

### T-03-12: bt-backend-llvm — 调试信息

- **ID**: `T-03-12`
- **状态**: TODO
- **估计**: 2 天
- **优先级**: P2
- **Sprint**: S-03-4

**描述**：生成 DWARF 调试信息。

**前置依赖**（DEP）：
- `T-02-06`

**验收标准**（TST）：
- [ ] emit_object 含 debug_info
- [ ] gdb/lldb 可调试生成的二进制

---

### T-03-13: 端到端测试 — 主流生态项目编译

- **ID**: `T-03-13`
- **状态**: TODO
- **估计**: 3 天
- **优先级**: P0
- **Sprint**: S-03-4

**验收标准**（TST）：
- [ ] serde/tokio/axum 项目编译通过
- [ ] `bt clippy` 在这些项目上运行正确

---

### T-03-14: Phase 3 回归测试

- **ID**: `T-03-14`
- **状态**: TODO
- **估计**: 1 天
- **优先级**: P1
- **Sprint**: S-03-4

**验收标准**（TST）：
- [ ] `cargo build/test/fmt/clippy --workspace` 通过
- [ ] 主流 Rust 生态项目编译通过

---

## Phase 3 可选扩展 — 详细 Task 分解

### T-03-E01: bt-policy — 许可策略 + 漏洞扫描

- **ID**: `T-03-E01`
- **来源**: F02 供应链安全
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选（Phase 3 完成后追加）

**描述**：实现许可协议策略和 RustSec 漏洞数据库集成。

**产出文件**：
- `crates/bt-policy/src/license.rs`
- `crates/bt-policy/src/vuln.rs`

**前置依赖**（DEP）：
- `T-02-E04`：bt-policy 来源策略

**验收标准**（TST）：
- [ ] `LicensePolicy`：允许/禁止许可 + 未知许可处理
- [ ] RustSec advisory-db 集成
- [ ] 漏洞严重度阈值过滤（Info/Low/Medium/High/Critical）
- [ ] `bt audit` 命令显示审计报告

---

### T-03-E02: bt-policy — SBOM 生成（CycloneDX + SPDX）

- **ID**: `T-03-E02`
- **来源**: F02 供应链安全
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现 SBOM 生成器，支持 CycloneDX 和 SPDX 双格式。

**产出文件**：
- `crates/bt-policy/src/sbom.rs`
- `crates/bt-policy/src/formats/cyclonedx.rs`
- `crates/bt-policy/src/formats/spdx.rs`

**前置依赖**（DEP）：
- `T-03-E01`

**验收标准**（TST）：
- [ ] CycloneDX v1.5 格式 SBOM 导出
- [ ] SPDX v2.3 格式 SBOM 导出
- [ ] 包含完整依赖树 + 版本 + 许可 + 哈希
- [ ] `bt sbom --format=cyclonedx --output=bom.json`

---

### T-03-E03: bt-policy — SLSA 证明 + 完整性验证

- **ID**: `T-03-E03`
- **来源**: F02 供应链安全
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：实现 SLSA 构建来源证明和依赖完整性验证。

**产出文件**：
- `crates/bt-policy/src/slsa.rs`
- `crates/bt-policy/src/integrity.rs`

**前置依赖**（DEP）：
- `T-03-E02`

**验收标准**（TST）：
- [ ] SLSA L1-L3 构建来源证明（in-toto attestation）
- [ ] SHA-256 校验和验证
- [ ] 完整性验证集成到 EventStore
- [ ] `bt attest --output=provenance.intoto.jsonl`

---

### T-03-E04: bt-toolchain — 链接器配置 + cargo config 生成

- **ID**: `T-03-E04`
- **来源**: F09 交叉编译环境管理
- **估计**: 5 天
- **优先级**: P3
- **Sprint**: 可选

**描述**：完善交叉编译的链接器自动配置和 cargo 配置生成。

**产出文件**：
- `crates/bt-toolchain/src/linker.rs` — 完善

**前置依赖**（DEP）：
- `T-02-E06`：bt-toolchain sysroot 下载

**验收标准**（TST）：
- [ ] 链接器自动检测：gcc/lld/mold
- [ ] 链接器包装脚本生成
- [ ] `.cargo/config.toml` 自动生成
- [ ] `bt build --target aarch64-unknown-linux-gnu` 完整工作流
