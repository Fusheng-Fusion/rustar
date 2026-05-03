# F02 — 软件供应链安全

> **优先级**：🔴 高 | **复杂度**：中 | **建议 Phase**：Phase 2-3 | **工期**：~5 周

---

## 1. 概述

软件供应链攻击逐年上升（如 `event-stream` 恶意包、`colors.js` 投毒），编译器作为软件构建的入口，天然适合承担供应链安全职责。本方案在 `bt-cargo` 依赖解析阶段注入**策略引擎 + SBOM 生成 + SLSA 证明 + 依赖完整性验证**。

### 核心理念

```
传统的编译器只关注"编译"，不关注"编译了什么"。
BlockType 作为编译器平台，应该知道：
  1. 编译了哪些依赖（SBOM）
  2. 依赖是否可信（策略引擎）
  3. 构建过程是否可复现（SLSA）
  4. 依赖内容是否被篡改（完整性验证）
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                      bt-policy crate                               │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    PolicyEngine                             │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │     │
│  │  │ SourcePolicy  │  │ LicensePolicy │  │ VersionPolicy │    │     │
│  │  │ (依赖来源)     │  │ (许可协议)     │  │ (版本范围)     │    │     │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │     │
│  │  ┌──────────────┐  ┌──────────────┐                     │     │
│  │  │ VulnPolicy    │  │ AuditPolicy   │                     │     │
│  │  │ (漏洞扫描)     │  │ (审计策略)     │                     │     │
│  │  └──────────────┘  └──────────────┘                     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    SBOM Generator                           │     │
│  │  - CycloneDX (OWASP 标准)                                 │     │
│  │  - SPDX (Linux Foundation 标准)                            │     │
│  │  - 包含：依赖树 + 版本 + 许可 + 哈希 + 来源                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    IntegrityVerifier                         │     │
│  │  - SHA-256 校验和验证                                      │     │
│  │  - crates.io 签名验证（未来）                               │     │
│  │  - git commit SHA 锁定                                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    SLSA Provenance                          │     │
│  │  - SLSA Level 1-3 构建来源证明                              │     │
│  │  - in-toto attestation 格式                               │     │
│  │  - 可导出为 JSON / DSSE 签名                               │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心类型定义

```rust
// crates/bt-policy/src/lib.rs

// ─── 策略引擎 ───

/// 编译策略配置
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BuildPolicy {
    /// 依赖来源策略
    pub source_policy: SourcePolicy,
    /// 许可协议策略
    pub license_policy: LicensePolicy,
    /// 版本策略
    pub version_policy: VersionPolicy,
    /// 漏洞策略
    pub vuln_policy: VulnPolicy,
    /// 审计策略（记录所有策略违反）
    pub audit_policy: AuditPolicy,
}

/// 依赖来源策略
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SourcePolicy {
    /// 允许的来源
    pub allowed_sources: Vec<SourceKind>,
    /// 禁止的来源
    pub blocked_sources: Vec<SourceKind>,
    /// 是否禁止 git 依赖
    pub block_git_deps: bool,
    /// 是否禁止 path 依赖
    pub block_path_deps: bool,
}

pub enum SourceKind {
    CratesIo,
    Git(String),           // 允许的 git 仓库 URL
    Path(PathBuf),         // 允许的本地路径
    CustomRegistry(String),// 自定义 registry URL
}

/// 许可协议策略
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LicensePolicy {
    /// 允许的许可列表
    pub allowed: Vec<String>,
    /// 禁止的许可列表
    pub forbidden: Vec<String>,
    /// 未知许可的处理方式
    pub on_unknown: PolicyAction,
}

/// 版本策略
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VersionPolicy {
    /// 是否要求版本锁定
    pub require_lockfile: bool,
    /// 是否禁止通配符版本
    pub block_wildcard: bool,
}

/// 漏洞策略
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VulnPolicy {
    /// 漏洞数据库 URL
    pub advisory_db_url: Option<String>,
    /// 严重度阈值：低于此等级的漏洞仅警告
    pub severity_threshold: Severity,
    /// 已知漏洞的处理方式
    pub on_vulnerability: PolicyAction,
}

pub enum Severity { Info, Low, Medium, High, Critical }

pub enum PolicyAction {
    /// 仅记录审计日志
    Audit,
    /// 发出警告
    Warn,
    /// 阻止编译
    Block,
}

// ─── SBOM 生成 ───

/// SBOM 生成器
pub struct SBOMGenerator;

impl SBOMGenerator {
    /// 生成 CycloneDX 格式 SBOM
    pub fn generate_cyclonedx(
        workspace: &CargoWorkspace,
        build_plan: &BuildPlan,
    ) -> Result<String, PolicyError>;

    /// 生成 SPDX 格式 SBOM
    pub fn generate_spdx(
        workspace: &CargoWorkspace,
        build_plan: &BuildPlan,
    ) -> Result<String, PolicyError>;
}

// ─── 完整性验证 ───

/// 依赖完整性验证器
pub struct IntegrityVerifier {
    /// 验证器列表
    verifiers: Vec<Box<dyn IntegrityCheck>>,
}

#[async_trait]
pub trait IntegrityCheck: Send + Sync {
    fn name(&self) -> &str;
    async fn verify(&self, dep: &CargoCrate) -> Result<IntegrityResult, PolicyError>;
}

pub struct IntegrityResult {
    pub passed: bool,
    pub details: String,
}

// ─── SLSA 证明 ───

/// 构建来源证明
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SLSAProvenance {
    pub builder: BuilderInfo,
    pub build_type: String,
    pub invocation: InvocationInfo,
    pub materials: Vec<MaterialInfo>,
    pub byproducts: Vec<ByproductInfo>,
    pub slsa_level: u32,
}

pub struct BuilderInfo {
    pub id: String,
    pub version: String,
}

pub struct InvocationInfo {
    pub config_source: ConfigSource,
    pub parameters: HashMap<String, String>,
    pub environment: HashMap<String, String>,
}

pub struct MaterialInfo {
    pub uri: String,
    pub digest: HashMap<String, String>,
}
```

### 2.3 策略配置文件

```toml
# blocktype-policy.toml（项目根目录）

[source]
allowed = ["crates-io", "git:github.com"]
blocked = ["git:unknown-registry.com"]
block_git_deps = false
block_path_deps = true

[license]
allowed = ["MIT", "Apache-2.0", "BSD-3-Clause", "MPL-2.0"]
forbidden = ["AGPL-3.0", "GPL-3.0"]
on_unknown = "warn"

[version]
require_lockfile = true
block_wildcard = true

[vulnerability]
advisory_db_url = "https://github.com/rustsec/advisory-db"
severity_threshold = "medium"
on_vulnerability = "block"

[audit]
output = "audit-report.json"
format = "json"
```

### 2.4 与 bt-cargo 的集成点

```rust
// bt-cargo 的 BuildPlanner 扩展

impl BuildPlanner {
    /// 设置策略引擎（可选）
    pub fn set_policy_engine(&mut self, engine: Arc<PolicyEngine>) {
        self.policy_engine = Some(engine);
    }

    /// 生成编译计划（增加策略检查步骤）
    pub async fn generate_plan(&self, workspace: &CargoWorkspace) -> Result<BuildPlan, CargoError> {
        // 1. 解析依赖图（现有逻辑）
        let deps = self.resolver.resolve(workspace)?;

        // 2. 策略检查（新增）
        if let Some(ref engine) = self.policy_engine {
            let violations = engine.check_all(&deps).await?;
            if !violations.is_empty() {
                // 根据 PolicyAction 决定是 warn 还是 block
                for v in &violations {
                    match v.action {
                        PolicyAction::Block => return Err(CargoError::PolicyViolation(v.clone())),
                        PolicyAction::Warn => warn!("{}", v.message),
                        PolicyAction::Audit => info!("{}", v.message),
                    }
                }
            }

            // 3. 完整性验证（新增）
            self.integrity_verifier.verify_all(&deps).await?;
        }

        // 4. 生成 SBOM（新增，可选）
        // ...

        Ok(BuildPlan { crates: deps, /* ... */ })
    }
}
```

### 2.5 CI/CD 集成

```yaml
# .github/workflows/blocktype-ci.yml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - run: bt build --policy=blocktype-policy.toml

  sbom:
    steps:
      - uses: actions/checkout@v4
      - run: bt sbom --format=cyclonedx --output=bom.json
      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: bom.json

  attest:
    steps:
      - run: bt attest --output=provenance.intoto.jsonl
      - run: bt attest --sign --key=${{ secrets.SIGSTORE_KEY }}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **策略引擎** | 基于 TOML 配置的依赖来源/许可/版本/漏洞策略 |
| C2 | **SBOM 生成** | CycloneDX + SPDX 双格式，可用于合规审计 |
| C3 | **完整性验证** | SHA-256 校验 + crates.io 签名验证 |
| C4 | **SLSA 证明** | L1-L3 构建来源证明（in-toto attestation） |
| C5 | **漏洞扫描** | 集成 RustSec advisory-db，编译时自动检测已知漏洞 |
| C6 | **许可合规** | 自动检测依赖的软件许可，阻止侵权依赖 |
| C7 | **审计日志** | 所有策略违反记录到 EventStore，可导出审计报告 |
| C8 | **CI 集成** | `bt sbom` / `bt attest` 子命令，适配 GitHub Actions |
| C9 | **策略模板** | 内置 enterprise/starter 等预设策略模板 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-policy` | 策略引擎 + SBOM + SLSA + 完整性验证 | ~3,500 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-cargo` | BuildPlanner 增加策略检查钩子 | 低（可选特性） |
| `bt-cli` | 新增 `--policy` / `sbom` / `attest` 子命令 | 低 |
| `bt-event-store` | 新增 PolicyViolation 事件类型 | 低 |

### CLI 变更

```bash
# ─── 策略模式 ───

bt build --policy=enterprise            # 使用预设的企业策略
bt build --policy=blocktype-policy.toml # 使用自定义策略文件
bt build --policy=off                   # 禁用策略检查

# ─── SBOM ───

bt sbom                                  # 生成 CycloneDX SBOM
bt sbom --format=spdx                    # SPDX 格式
bt sbom --output=bom.json                # 输出到文件

# ─── 来源证明 ───

bt attest                                # 生成 SLSA 来源证明
bt attest --sign                         # 签名（需配置密钥）

# ─── 安全检查 ───

bt audit                                 # 审计依赖安全检查
bt audit --json                          # JSON 格式审计报告

# ─── 策略模板 ───

bt policy init                           # 生成默认策略文件
bt policy init --template=enterprise     # 使用企业模板
bt policy check                          # 检查当前项目是否符合策略
```

---

## 5. 分阶段实施

### Phase 2 扩展（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-policy` crate: PolicyEngine + 策略配置解析 | `bt build --policy=off` 可用 |
| 2 | 来源策略 + 版本策略 + bt-cargo 集成 | 阻止未授权依赖 |

### Phase 3 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 3 | 许可策略 + 漏洞扫描（RustSec） | `bt audit` 显示已知漏洞 |
| 4 | SBOM 生成（CycloneDX + SPDX） | `bt sbom` 输出标准 SBOM |
| 5 | SLSA 证明 + 完整性验证 + EventStore | `bt attest` 输出来源证明 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 策略误报导致合法项目无法编译 | 中 | 高 | PolicyAction::Warn 作为默认，允许项目覆盖 |
| 漏洞数据库更新延迟 | 中 | 中 | 支持 `bt audit --update` 手动更新 + 定时自动更新 |
| SBOM 导出版本与标准不完全一致 | 低 | 中 | 严格遵循 CycloneDX v1.5 / SPDX v2.3 规范 |
| 策略配置复杂度增加使用门槛 | 中 | 低 | 提供预设模板（starter/enterprise），一行命令初始化 |
