# F04 — 编译时 Fuzzing / 变异测试

> **优先级**：🟡 中 | **复杂度**：中 | **建议 Phase**：Phase 5+ | **工期**：~5 周

---

## 1. 概述

传统 fuzzing 是运行时的——编译后通过对程序输入进行变异来发现 bug。本方案将 fuzzing 前移到**编译时**，利用编译器对代码结构和控制流的完整理解，自动生成测试用例、验证优化 Pass 的正确性、以及检查测试覆盖率。

### 核心理念

```
运行时 Fuzzing：    编译 → [已部署的程序] → 随机输入 → 检测崩溃
编译时 Fuzzing：    [源码] → 语义保持变异 → 双路径编译 → 输出对比 → 检测优化错误
                 ↑ 编译器知道代码结构、控制流、类型信息，可以做更智能的变异
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                       bt-fuzz crate                                │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    IRMutator                                │     │
│  │  - 语义保持的 IR 变换（等价变异）                           │     │
│  │  - 变异策略：常量替换/操作交换/分支翻转/死代码注入           │     │
│  │  - IR 级变异 vs 源码级变异                                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    DiffTestEngine                            │     │
│  │  - 双路径对比：原始 IR vs 变异后 IR                       │     │
│  │  - Pass 等价性验证：优化前后语义是否一致                   │     │
│  │  - 跨后端对比：LLVM vs Cranelift 输出是否等价              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    FuzzHarnessGenerator                      │     │
│  │  - 从 crate 自动提取 fuzz harness 模板                     │     │
│  │  - 识别输入边界（网络输入 / 文件解析 / API 参数）          │     │
│  │  - 输出 libFuzzer / AFL 兼容的 harness                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    MutationCoverage                         │     │
│  │  - 变异测试覆盖率（Mutation Score）                        │     │
│  │  - 检测"未覆盖"的代码路径                                  │     │
│  │  - 生成覆盖率报告（Dashboard 集成）                         │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心类型定义

```rust
// crates/bt-fuzz/src/lib.rs

// ─── IR 变异器 ───

/// IR 变异器 — 对 IRModule 进行语义保持变换
pub struct IRMutator {
    /// 随机数生成器
    rng: Arc<Mutex<StdRng>>,
    /// 变异策略权重
    strategy_weights: HashMap<MutationStrategy, f64>,
}

/// 变异策略
pub enum MutationStrategy {
    /// 常量值替换（如 42 → 43 / 0 → 1）
    ConstantSubstitution,
    /// 算术操作交换（如 a + b → b + a）
    CommutativeSwap,
    /// 条件分支翻转（如 if cond → if !cond + 交换分支）
    BranchFlip,
    /// 死代码注入（插入不会影响结果的计算）
    DeadCodeInsertion,
    /// GEP 索引偏移
    GEPOffsetShift,
    /// 函数调用参数乱序（对交换律操作）
    CallArgReorder,
    /// 类型不变的值变换（如 i32: x + 0 → x）
    IdentityTransform,
    /// 循环展开/折叠
    LoopUnrollFold,
}

impl IRMutator {
    /// 对 IRModule 执行一次语义保持变异
    pub fn mutate(&self, module: &IRModule) -> Result<IRModule, FuzzError>;

    /// 生成 N 个变异版本
    pub fn mutate_many(&self, module: &IRModule, n: usize) -> Result<Vec<IRModule>, FuzzError>;

    /// 随机选择变异位置
    fn select_mutation_point(&self, module: &IRModule) -> MutationPoint;

    /// 在指定位置应用变异
    fn apply_mutation(&self, module: &mut IRModule, point: MutationPoint, strategy: MutationStrategy);
}

// ─── 差异测试引擎 ───

/// 差异测试 — 验证两个 IRModule 的编译结果是否等价
pub struct DiffTestEngine {
    /// 后端 A（通常是优化后的）
    backend_a: Arc<dyn Backend>,
    /// 后端 B（通常是优化前的）
    backend_b: Arc<dyn Backend>,
    /// 等价性检查器
    equivalence_checker: IREquivalenceChecker,
}

impl DiffTestEngine {
    /// 执行差异测试
    ///
    /// 1. 用 backend_a 编译 original IR
    /// 2. 用 backend_b 编译 mutated IR
    /// 3. 对比输出等价性
    pub async fn diff_test(
        &self,
        original: &IRModule,
        mutated: &IRModule,
    ) -> Result<DiffTestResult, FuzzError>;

    /// Pass 等价性测试：验证 Pass 应用前后语义保持不变
    pub async fn pass_equivalence_test(
        &self,
        module: &IRModule,
        pass: &dyn Pass,
    ) -> Result<PassEquivalenceResult, FuzzError>;
}

pub struct DiffTestResult {
    pub equivalent: bool,
    pub original_output: BackendOutput,
    pub mutated_output: BackendOutput,
    pub differences: Vec<DiffDetail>,
    pub mutation_applied: MutationStrategy,
}

// ─── Fuzz Harness 生成器 ───

/// Fuzz harness 自动生成
pub struct FuzzHarnessGenerator;

impl FuzzHarnessGenerator {
    /// 从 IRModule 提取可 fuzz 的接口
    pub fn extract_fuzzable_interfaces(
        &self,
        module: &IRModule,
    ) -> Vec<FuzzableInterface>;

    /// 生成 libFuzzer 兼容的 harness
    pub fn generate_libfuzzer_harness(
        &self,
        interface: &FuzzableInterface,
    ) -> String;

    /// 生成 AFL 兼容的 harness
    pub fn generate_afl_harness(
        &self,
        interface: &FuzzableInterface,
    ) -> String;
}

pub struct FuzzableInterface {
    pub function_name: String,
    pub param_types: Vec<IRType>,
    pub return_type: IRType,
    pub input_constraints: Vec<String>, // 参数约束（如 "non-null pointer"）
}

// ─── 变异测试覆盖率 ───

/// 变异测试覆盖率报告
#[derive(Debug, Clone, Serialize)]
pub struct MutationCoverageReport {
    pub total_mutants: usize,
    pub killed: usize,         // 被测试捕获的变异数
    pub survived: usize,       // 未被测试捕获的变异数
    pub mutation_score: f64,   // killed / total
    pub coverage_by_module: HashMap<String, f64>,
    pub strongest_mutants: Vec<SurvivedMutant>, // 最高质量的存活变异
    pub uncovered_regions: Vec<SourceLocation>, // 未覆盖的代码区域
}
```

### 2.3 工作流程

```
bt fuzz <crate>
  │
  ├── 1. 编译 crate 为 IRModule
  │
  ├── 2. IRMutator: 生成 N 个变异 IRModule
  │     ├── Mutant 1: ConstantSubstitution(fn foo, line 42)
  │     ├── Mutant 2: BranchFlip(fn bar, block 3)
  │     ├── Mutant 3: DeadCodeInsertion(fn baz, block 7)
  │     └── ...
  │
  ├── 3. DiffTestEngine: 逐一对每个 mutant 执行差异测试
  │     ├── Mutant 1: equivalent ✅（正确）
  │     ├── Mutant 2: equivalent ✅
  │     └── Mutant 3: NOT equivalent ❌（发现 Pass 错误！）
  │
  ├── 4. FuzzHarnessGenerator: 从入口函数生成 harness
  │
  └── 5. 输出报告
        ├── Pass 等价性: 3/3 passed
        ├── Fuzz harness: 生成到 bt-fuzz-harness/
        └── 发现: 1 个 Pass 等价性违反（Pass: dce, Function: baz）
```

### 2.4 与 bt-passes 的集成

```rust
// bt-fuzz 自动发现并测试所有已注册的 Pass

impl FuzzTester {
    /// 对 PassManager 中所有 Pass 执行等价性测试
    pub async fn test_all_passes(
        &self,
        pass_manager: &PassManager,
        test_modules: &[IRModule],
    ) -> Vec<PassEquivalenceReport> {

        let mut reports = vec![];

        for pass in pass_manager.registered_passes() {
            if !pass.category().is_transformation() {
                continue;  // 只测试变换类 Pass
            }

            for module in test_modules {
                let result = self.diff_engine.pass_equivalence_test(module, pass.as_ref()).await;
                reports.push(PassEquivalenceReport {
                    pass_name: pass.name().to_string(),
                    module: module.name.clone(),
                    passed: result.equivalent,
                    details: result.differences,
                });
            }
        }

        reports
    }
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **语义保持变异** | 8 种变异策略，自动保证语义等价 |
| C2 | **差异测试引擎** | 跨后端对比验证 Pass 正确性 |
| C3 | **Pass 等价性测试** | `bt fuzz --pass=dce` 验证特定 Pass |
| C4 | **Fuzz Harness 生成** | 从源码自动提取 fuzz 入口 |
| C5 | **变异测试覆盖率** | Mutation Score 报告，发现测试盲区 |
| C6 | **跨后端对比** | LLVM / Cranelift / rustc 原生 输出等价性 |
| C7 | **边界值探索** | 自动识别参数边界（数组长度、枚举变体） |
| C8 | **CI 集成** | `bt fuzz --ci` 模式下运行简化版（少变异数） |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-fuzz` | IR 变异器 + 差异测试 + FuzzHarness 生成 + 覆盖率 | ~3,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-pass-manager` | 暴露 `registered_passes()` 给 bt-fuzz 使用 | 低 |
| `bt-cli` | 新增 `fuzz` 子命令 | 低 |
| `bt-event-store` | 新增 Fuzz 事件类型 | 低 |

### CLI 变更

```bash
# ─── Fuzzing ───

bt fuzz                                  # 对整个项目执行编译时 fuzzing
bt fuzz --mutants=1000                    # 生成 1000 个变异
bt fuzz --pass=dce                        # 只测试 DCE Pass 的等价性

bt fuzz --backend=cranelift               # 使用 Cranelift 作为参考后端
bt fuzz --backend=llvm                    # 使用 LLVM 作为参考后端
bt fuzz --cross-backend                   # 跨后端对比验证

# ─── 变异测试 ───

bt fuzz --mutation-score                  # 计算变异测试覆盖率
bt fuzz --coverage-html=report.html       # 生成 HTML 覆盖率报告

# ─── Fuzz Harness ───

bt fuzz --harness                         # 生成 fuzz harness
bt fuzz --harness-dir=fuzz/               # 指定输出目录
bt fuzz --harness-target=parse_input      # 指定目标函数

# ─── CI 模式 ───

bt fuzz --ci                              # 简化版（快速执行）
bt fuzz --ci --mutants=100                # CI 模式下只测 100 个变异
```

---

## 5. 分阶段实施

### Phase 5 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-fuzz` crate: IRMutator + 4 种基础变异策略 | 可对 IRModule 执行语义保持变异 |
| 2 | DiffTestEngine + Pass 等价性测试 | 可验证优化 Pass 是否正确 |
| 3 | CLI 集成 + 基础测试 | `bt fuzz --pass=dce` 可用 |

### 扩展阶段（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 4 | FuzzHarnessGenerator + 变异覆盖率 | 自动生成 fuzz harness |
| 5 | CI 模式 + EventStore + Dashboard | CI 集成 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 语义保持变异可能引入非等价代码 | 中 | 中 | 双路径编译输出对比验证 |
| 差异测试有 false positive | 低 | 中 | 后端输出等价性检查仅在相同后端下进行 |
| 大量变异导致编译时间过长 | 高 | 中 | `--mutants` 控制 + CI 模式减量 |
| Fuzz Harness 提取不准确 | 中 | 低 | 以模板形式输出，开发者可手动调整 |
