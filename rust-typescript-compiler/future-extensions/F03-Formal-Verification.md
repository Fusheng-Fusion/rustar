# F03 — 编译时形式化验证集成

> **优先级**：🟡 中 | **复杂度**：高 | **建议 Phase**：Phase 5+ | **工期**：~8 周

---

## 1. 概述

Rust 已被广泛应用于操作系统、嵌入式、自动驾驶等安全关键领域。在这些场景中，仅靠 type system 和 borrow checker 无法证明程序的全部安全性。本方案将**形式化验证**（formal verification）作为编译器的一等能力，在 BTIR 层设计合约元数据，并与 Kani/Prusti 等验证工具集成。

### 核心理念

```
传统编译器：
  源码 → [typeck + borrowck] → 机器码（确保内存安全，但不确保逻辑正确）

BlockType + 形式化验证：
  源码 → [合约标注] → [BTIR 层合约验证] → [Post-condition 检查] → 机器码（确保逻辑正确）
                  ↑                    ↑
             Kani/Prusti 输入    编译时静态断言
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                      bt-contract crate                             │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    ContractAnnotation                        │     │
│  │  - Precondition / Postcondition / Invariant                 │     │
│  │  - bt_rust Dialect 操作码扩展                              │     │
│  │  - 序列化为 IR 元数据                                      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    ContractVerifierPass                      │     │
│  │  - IR 级静态断言检查                                       │     │
│  │  - SMT 求解器集成 (Z3 / Bitwuzla)                         │     │
│  │  - 符号执行引擎                                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    VerifierBridge                            │     │
│  │  - Kani Rust Verifier 输出对接                              │     │
│  │  - Prusti 合约格式对接                                      │     │
│  │  - 验证结果 → CompilerEvent                                 │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

IR 层合约标注：
  IRFunction.contract: Option<ContractAnnotation>

合约检查管线：
  Lower → [ContractSpecPass: 提取合约元数据] → [ContractVerifierPass: 符号执行验证] → Optimize → Codegen
                                  ↓
                         验证结果写入 EventStore
```

### 2.2 核心类型定义

```rust
// crates/bt-contract/src/lib.rs

// ─── 合约标注 ───

/// 函数合约标注
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContractAnnotation {
    /// 前置条件（函数入口处必须满足）
    pub preconditions: Vec<ContractExpr>,
    /// 后置条件（函数退出时必须满足）
    pub postconditions: Vec<ContractExpr>,
    /// 循环不变量
    pub loop_invariants: Vec<LoopInvariant>,
    /// 类型不变量（整个结构体生命周期内必须满足）
    pub type_invariants: Vec<ContractExpr>,
}

/// 合约表达式（以 IRValue 为操作数的逻辑表达式）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ContractExpr {
    // ─── 基本比较 ───
    Eq { lhs: IRValueId, rhs: IRValueId },
    Neq { lhs: IRValueId, rhs: IRValueId },
    Lt { lhs: IRValueId, rhs: IRValueId },
    Lte { lhs: IRValueId, rhs: IRValueId },
    Gt { lhs: IRValueId, rhs: IRValueId },
    Gte { lhs: IRValueId, rhs: IRValueId },

    // ─── 逻辑组合 ───
    And { left: Box<ContractExpr>, right: Box<ContractExpr> },
    Or { left: Box<ContractExpr>, right: Box<ContractExpr> },
    Not { inner: Box<ContractExpr> },
    Implies { premise: Box<ContractExpr>, conclusion: Box<ContractExpr> },

    // ─── 量化 ───
    ForAll {
        var: IRValueId,
        range: IRRange,
        body: Box<ContractExpr>,
    },
    Exists {
        var: IRValueId,
        range: IRRange,
        body: Box<ContractExpr>,
    },

    // ─── 特殊 ───
    Old(Box<ContractExpr>),           // 函数入口时的值（后置条件中使用）
    Result(IRValueId),                // 函数返回值
    PanicFree,                         // 保证不会 panic
    NoOverflow,                        // 算术运算不会溢出
}

/// 循环不变量
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LoopInvariant {
    pub invariant: ContractExpr,
    pub loop_block: IRBlockId,
}

// ─── 验证 Pass ───

/// 合约验证 Pass
pub struct ContractVerifierPass {
    /// SMT 求解器
    solver: Box<dyn SMTSolver>,
    /// 验证超时
    timeout: Duration,
    /// 最大路径数（防止路径爆炸）
    max_paths: usize,
}

#[async_trait]
impl Pass for ContractVerifierPass {
    fn name(&self) -> &str { "contract_verifier" }
    fn category(&self) -> PassCategory { PassCategory::Verification }

    async fn run(&self, module: &mut IRModule, ctx: &PassContext) -> PassResult {
        let mut verified = 0u64;
        let mut failed = 0u64;

        for func in &module.functions {
            if let Some(ref contract) = func.contract {
                match self.verify_function(func, contract, ctx).await {
                    VerificationResult::Verified => verified += 1,
                    VerificationResult::Violated(report) => {
                        failed += 1;
                        ctx.diagnostics.push(Diagnostic::error(
                            format!("合约违反: {}", report),
                        ));
                    }
                    VerificationResult::Timeout => {
                        ctx.diagnostics.push(Diagnostic::warning(
                            format!("合约验证超时: {}", func.name),
                        ));
                    }
                }
            }
        }

        PassResult {
            modified: false,  // 验证 Pass 不修改 IR
            metrics: PassMetrics::default()
                .with_counter("verified", verified)
                .with_counter("failed", failed),
            diagnostics: ctx.diagnostics.clone(),
        }
    }
}

// ─── SMT 求解器 trait ───

/// SMT 求解器抽象
#[async_trait]
pub trait SMTSolver: Send + Sync {
    fn name(&self) -> &str;
    /// 检查一组表达式是否可满足
    async fn check_satisfiability(&self, exprs: &[SMTExpr]) -> Result<SMTResult, SolverError>;
    /// 获取满足条件的模型
    async fn get_model(&self, exprs: &[SMTExpr]) -> Result<Option<Model>, SolverError>;
}

/// 验 Z3 求解器实现
pub struct Z3Solver {
    context: Z3Context,
    timeout: Duration,
}

/// 符号执行引擎
pub struct SymbolicExecutor {
    solver: Box<dyn SMTSolver>,
    max_paths: usize,
    max_depth: usize,
}

impl SymbolicExecutor {
    /// 对 IRFunction 执行符号执行
    pub async fn execute(&self, func: &IRFunction, contract: &ContractAnnotation)
        -> Result<Vec<SymbolicPath>, ExecutorError>;
}

// ─── 验证结果 ───

pub enum VerificationResult {
    Verified,
    Violated(VerificationReport),
    Timeout,
}

pub struct VerificationReport {
    pub function: String,
    pub violated_condition: String,
    pub counterexample: Option<Counterexample>,
    pub trace: Vec<SourceLocation>,
}
```

### 2.3 bt_rust Dialect 合约操作码

```
bt_rust 新增操作码 (238-239)：

| Opcode | 名称 | 说明 | 降级目标 |
|--------|------|------|---------|
| 238 | `contract_assert` | 编译时静态断言 | 编译期检查 → 无运行时指令 |
| 239 | `contract_assume` | 假设满足条件（给验证器提示） | 编译期检查 → 无运行时指令 |
```

### 2.4 与 Kani/Prusti 的集成

```rust
// crates/bt-contract/src/verifier_bridge.rs

/// 验证工具桥接
pub struct VerifierBridge {
    kani_bridge: Option<KaniBridge>,
    prusti_bridge: Option<PrustiBridge>,
}

/// Kani Rust Verifier 桥接
pub struct KaniBridge;

impl KaniBridge {
    /// 从 Kani 的输出提取验证结果
    pub fn parse_kani_output(
        &self,
        kani_output: &str,
        source_map: &SourceManager,
    ) -> Vec<VerificationReport>;

    /// 将 BTIR 合约转换为 Kani 的合约格式
    pub fn contract_to_kani(
        &self,
        contract: &ContractAnnotation,
    ) -> String;
}

/// Prusti 桥接
pub struct PrustiBridge;

impl PrustiBridge {
    /// 从 Prusti 的输出提取验证结果
    pub fn parse_prusti_output(
        &self,
        prusti_output: &str,
    ) -> Vec<VerificationReport>;
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **IR 级合约标注** | Precondition/Postcondition/Invariant 作为 IR 一等元数据 |
| C2 | **编译时静态断言** | `contract_assert` 指令，编译期验证而非运行时 |
| C3 | **SMT 求解器集成** | Z3 + Bitwuzla 双求解器，可插拔 SMTSolver trait |
| C4 | **符号执行引擎** | 路径敏感分析，支持路径爆炸控制（max_paths + max_depth） |
| C5 | **Kani 集成** | 解析 Kani 验证结果 → BlockType CompilerEvent |
| C6 | **Prusti 集成** | 兼容 Prusti 合约标注格式 |
| C7 | **反例展示** | 违反合约时提供具体的 counterexample |
| C8 | **可验证覆盖度** | `bt verify --coverage` 显示已验证的代码路径占比 |
| C9 | **可选启用** | 合约验证不影响正常编译，仅在 `--verify` 模式下运行 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-contract` | 合约标注 + 验证 Pass + SMT 求解器 + 验证桥接 | ~4,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-ir` | `IRFunction` 增加 `contract: Option<ContractAnnotation>` | 低 |
| `bt-dialect-rust` | 新增 `contract_assert` / `contract_assume` 操作码 | 低 |
| `bt-passes` | PassManager 支持 Verification 类别 Pass | 无（已支持） |
| `bt-cli` | 新增 `verify` 子命令 | 低 |

### CLI 变更

```bash
# ─── 验证模式 ───

bt verify                              # 验证合约标注
bt verify --solver=z3                   # 使用 Z3 求解器
bt verify --solver=bitwuzla             # 使用 Bitwuzla 求解器
bt verify --timeout=30s                 # 验证超时 30 秒
bt verify --max-paths=1000              # 最大路径数

bt verify --coverage                    # 显示验证覆盖度
bt verify --kani                        # 使用 Kani 验证（需安装 kani）
bt verify --prusti                      # 使用 Prusti 验证（需安装 prusti）

bt build --verify                       # 编译 + 合约验证一步完成

# ─── 合约标注辅助 ───

bt contract init                        # 在当前项目创建合约模板
bt contract explain                     # 显示合约语言参考
```

---

## 5. 分阶段实施

### Phase 5 扩展（4 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-contract` crate: ContractAnnotation + 序列化 | 合约元数据可在 IR 中存储 |
| 2 | bt_rust Dialect 合约操作码 + 编译时静态断言 Pass | `contract_assert` 编译期检查 |
| 3 | SMT 求解器 trait + Z3 集成 | `bt verify` 可运行约束求解 |
| 4 | 符号执行引擎 + 反例生成 | 合约违反时可显示反例 |

### 扩展阶段（4 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 5-6 | Kani/Prusti 输出对接 | 兼容现有 Rust 验证生态 |
| 7 | 验证覆盖度统计 | `bt verify --coverage` |
| 8 | 端到端集成 + 示例文档 | 安全关键项目可使用 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| SMT 求解时间过长 | 高 | 中 | 超时机制 + 路径数限制 + 可选启用 |
| 路径爆炸导致验证不完整 | 高 | 中 | SymbolicExecutor 的 max_paths/max_depth 配置 |
| Z3 求解器集成难度 | 中 | 中 | 使用 `z3-rs` crate + 子进程调用 |
| Kani/Prusti 输出格式变更 | 中 | 低 | 版本锁定 + 适配层隔离 |
