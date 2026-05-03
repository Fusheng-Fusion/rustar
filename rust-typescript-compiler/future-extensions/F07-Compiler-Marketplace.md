# F07 — Compiler Marketplace（编译器即平台）

> **优先级**：🟡 中 | **复杂度**：中 | **建议 Phase**：Phase 5+ | **工期**：~5 周

---

## 1. 概述

当前 BlockType 已经定义了 **WASM 插件系统**（WIT 接口 + 沙箱隔离）、**运行时 Dialect 注册**、**RESTful API**——这些基础设施使得 BlockType 具备了"编译器平台"的雏形。本方案在此基础上构建**插件生态**：定义标准插件包格式、注册表协议、版本管理、一键安装体验，让社区可以分发和发现 Dialect / Pass / Lint 规则 / AI Provider。

### 核心理念

```
编译器不再是一个"被安装的工具"，而是一个"可以扩展的平台"。
类似于 VS Code 的 Extension Marketplace，但针对编译器领域。
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                    bt-plugin-index crate                           │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                PluginRegistry                               │     │
│  │  - 本地已安装插件列表                                     │     │
│  │  - 插件版本管理（semver）                                  │     │
│  │  - 插件依赖解析                                           │     │
│  ├──────────────────────────────────────────────────────────┤     │
│  │                RemoteRegistryClient                          │     │
│  │  - 与注册表服务器通信                                     │     │
│  │  - 搜索/查询/安装/更新                                    │     │
│  │  - 验证插件签名                                            │     │
│  ├──────────────────────────────────────────────────────────┤     │
│  │                PluginPackage                                 │     │
│  │  - .btpkg 格式定义                                        │     │
│  │  - 包元数据 + WASM 二进制 + WIT 接口                       │     │
│  │  - 签名 + 哈希验证                                         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │            bt-plugin-registry-server (可选)                  │     │
│  │  - 注册表服务器实现（axum）                                │     │
│  │  - 插件上传/审核/分发                                      │     │
│  │  - 下载统计 + 评分系统                                     │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

插件类型：
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐
  │ Dialect     │  │ Pass        │  │ Lint Rule   │  │ AI Provider  │
  │ 插件         │  │ 插件         │  │ 插件         │  │ 插件          │
  └─────────────┘  └─────────────┘  └─────────────┘  └──────────────┘
```

### 2.2 核心类型定义

```rust
// crates/bt-plugin-index/src/lib.rs

// ─── 插件包定义 ───

/// 插件包类型
pub enum PluginKind {
    /// Dialect 扩展
    Dialect,
    /// IR Pass
    Pass,
    /// Lint 规则
    LintRule,
    /// AI Provider
    AIProvider,
}

/// 插件包元数据
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginPackage {
    // ─── 包标识 ───
    pub name: String,
    pub version: semver::Version,
    pub kind: PluginKind,
    pub description: String,
    pub authors: Vec<String>,
    pub license: String,
    pub repository: Option<String>,
    pub homepage: Option<String>,

    // ─── 依赖 ───
    pub bt_version_req: semver::VersionReq,  // 兼容的 BlockType 版本
    pub dependencies: Vec<PluginDependency>,

    // ─── 文件 ───
    pub wasm_blob_sha256: String,
    pub wasm_blob_url: Option<String>,     // 远程 URL（注册表用）
    pub wasm_blob: Option<Vec<u8>>,        // 本地缓存

    // ─── 签名 ───
    pub signature: Option<String>,
    pub signer: Option<String>,

    // ─── WIT 接口 ───
    pub wit_interfaces: Vec<String>,       // 实现的 WIT 接口列表
}

pub struct PluginDependency {
    pub name: String,
    pub version_req: semver::VersionReq,
}

// ─── 插件注册表客户端 ───

/// 注册表客户端
pub struct RegistryClient {
    registry_url: String,
    client: reqwest::Client,
    local_registry: LocalPluginRegistry,
}

impl RegistryClient {
    /// 搜索插件
    pub async fn search(&self, query: &str, kind: Option<PluginKind>) -> Result<Vec<PluginSummary>>;

    /// 安装插件（下载 + 验证 + 注册）
    pub async fn install(&mut self, name: &str, version: Option<&str>) -> Result<PluginPackage>;

    /// 卸载插件
    pub async fn uninstall(&mut self, name: &str) -> Result<()>;

    /// 更新所有已安装插件
    pub async fn update_all(&mut self) -> Result<Vec<PluginUpdateResult>>;

    /// 列出已安装插件
    pub fn list_installed(&self) -> Vec<PluginPackage>;

    /// 验证插件签名
    pub fn verify_signature(&self, pkg: &PluginPackage) -> Result<bool>;
}

// ─── 插件清单文件 ───

/// blocktype-plugins.toml（项目级插件配置）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginsConfig {
    /// 项目依赖的插件
    pub plugins: Vec<PluginPin>,

    /// 自定义注册表
    #[serde(default)]
    pub registries: Vec<RegistryConfig>,

    /// 本地插件路径
    #[serde(default)]
    pub local_plugins: Vec<PathBuf>,
}

pub struct PluginPin {
    pub name: String,
    pub version: String,           // "1.2.3" / ">=1.0,<2.0"
    pub source: Option<String>,    // registry URL / local path
}

pub struct RegistryConfig {
    pub name: String,
    pub url: String,
}
```

### 2.3 插件包格式 (.btpkg)

```
my-dialect-1.0.0.btpkg/
├── metadata.toml           # 插件元数据（name/version/kind/deps）
├── interface.wit           # WIT 接口定义
├── plugin.wasm             # WASM 二进制
├── README.md               # 使用说明
├── LICENSE                 # 许可协议
├── signature.sig           # 签名
└── examples/               # 使用示例
    └── example.rs
```

### 2.4 与现有插件的集成

```rust
// bt-plugin-host 扩展

impl PluginHost {
    /// 从 PluginPackage 加载插件
    pub async fn load_from_package(&mut self, pkg: &PluginPackage) -> Result<(), PluginError> {
        // 1. 验证 WASM 哈希
        let actual_hash = sha256(&pkg.wasm_blob);
        if actual_hash != pkg.wasm_blob_sha256 {
            return Err(PluginError::HashMismatch);
        }

        // 2. 验证签名（如果配置了验证）
        if self.config.verify_signatures {
            self.registry_client.verify_signature(pkg)?;
        }

        // 3. 加载 WASM 到沙箱
        let instance = self.wasmtime_engine
            .load_module(&pkg.wasm_blob, &pkg.wit_interfaces)?;

        // 4. 注册到对应 Registry
        match pkg.kind {
            PluginKind::Dialect => {
                let dialect = self.instantiate_dialect(instance)?;
                self.dialect_registry.register(dialect)?;
            }
            PluginKind::Pass => {
                let pass = self.instantiate_pass(instance)?;
                self.pass_manager.register(pass)?;
            }
            PluginKind::LintRule => {
                let rule = self.instantiate_lint(instance)?;
                // 注入到 bt-clippy-integration
            }
            PluginKind::AIProvider => {
                let provider = self.instantiate_ai_provider(instance)?;
                self.ai_orchestrator.register_provider(provider)?;
            }
        }

        // 5. 记录到本地注册表
        self.local_registry.add(pkg.clone())?;

        Ok(())
    }
}
```

### 2.5 OpenAPI 自动生成

```rust
// bt-api 扩展

impl ApiServer {
    /// 自动从 axum 路由生成 OpenAPI 规范
    pub fn generate_openapi_spec(&self) -> openapiv3::OpenAPI {
        // 使用 utoipa / aide 自动生成
        // 覆盖所有 /api/v1/* 端点
        // 包括：请求/响应 schema、错误码、认证方式
    }
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **插件包格式** | `.btpkg` 标准包定义，包含元数据 + WASM + WIT |
| C2 | **注册表客户端** | `bt plugin install/search/update` 命令 |
| C3 | **插件签名验证** | 基于 sigstore / GPG 的插件签名 |
| C4 | **版本管理** | semver 兼容，依赖解析，升级/降级 |
| C5 | **四种插件类型** | Dialect / Pass / Lint / AI Provider |
| C6 | **项目级插件锁** | `blocktype-plugins.toml` + `blocktype-plugins.lock` |
| C7 | **注册表服务器** | 可选自托管，支持审核流程 |
| C8 | **OpenAPI 自动生成** | 从 axum router 自动生成 API 文档 |
| C9 | **Compiler-as-a-Service** | `bt server --public` 启动公共编译服务 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-plugin-index` | 插件注册表客户端 + 包格式 + 版本管理 | ~2,500 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-plugin-host` | 新增 `load_from_package()` | 中 |
| `bt-api` | 新增注册表 API 端点 + OpenAPI 生成 | 低 |
| `bt-cli` | 新增 `plugin` 子命令 | 低 |

### CLI 变更

```bash
# ─── 插件管理 ───

bt plugin search <query>                # 搜索插件
bt plugin install <name>                # 安装插件
bt plugin install <name>@1.2.0          # 安装指定版本
bt plugin uninstall <name>              # 卸载插件
bt plugin update                        # 更新所有插件
bt plugin list                          # 列出已安装插件
bt plugin info <name>                   # 查看插件详情

# ─── 插件开发 ───

bt plugin init                          # 初始化插件项目骨架
bt plugin build                         # 构建插件为 .btpkg
bt plugin pack                          # 打包插件
bt plugin publish                       # 发布到注册表

# ─── 注册表管理 ───

bt plugin registry add <name> <url>     # 添加自定义注册表
bt plugin registry remove <name>        # 移除注册表
bt plugin registry list                 # 列出注册表

# ─── Compiler-as-a-Service ───

bt server                               # 启动编译器服务
bt server --public                      # 公开服务（需要认证配置）
```

---

## 5. 分阶段实施

### Phase 5 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `.btpkg` 格式定义 + PluginPackage 类型 | 可构建和解析插件包 |
| 2 | 本地注册表 + 安装/卸载/列表 | `bt plugin install/list/uninstall` 可用 |
| 3 | `bt-plugin-host` 集成 + WASM 加载验证 | 可通过 WASM 加载 Dialect/Pass 插件 |

### 扩展阶段（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 4 | 远程注册表客户端 + 搜索/更新 | `bt plugin search/update` 可用 |
| 5 | 签名验证 + OpenAPI 生成 | 插件签名验证 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 插件安全风险（恶意插件） | 中 | 高 | WASM 沙箱 + 签名验证 + 审核流程 |
| 插件 API 稳定性 | 中 | 中 | 严格的 semver + bt_version_req 约束 |
| 注册表服务器运维成本 | 低 | 低 | 可选自托管，提供参考实现 |
