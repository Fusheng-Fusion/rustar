# F09 — 交叉编译环境管理

> **优先级**：🟡 中 | **复杂度**：低 | **建议 Phase**：Phase 3 | **工期**：~3 周

---

## 1. 概述

Rust 交叉编译的痛点众所周知——需要手动安装目标平台的 sysroot、标准库、链接器，配置繁琐且容易出错。本方案将 `bt-std-bridge` 扩展为完整的**交叉编译工具链管理器**，`bt target` 一行命令即可安装和管理任意目标的交叉编译环境。

### 核心体验

```
# 当前 Rust 交叉编译:
rustup target add aarch64-unknown-linux-gnu  # 安装 std
# 还要手动装: aarch64-linux-gnu-gcc 链接器
# 还要配置: .cargo/config.toml
# 还要解决: openssl-sys 等 C 依赖的交叉编译

# BlockType 方案:
bt target add aarch64-unknown-linux-gnu  # 一步到位（安装全部）
bt build --target aarch64-unknown-linux-gnu  # 直接编译
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     bt-toolchain crate                              │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    ToolchainManager                          │     │
│  │  - 目标平台管理（安装/卸载/列表/更新）                      │     │
│  │  - 自动检测已安装的 toolchain                               │     │
│  │  - 与 rustup 集成（复用已有安装）                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    SysrootProvider                          │     │
│  │  - 自动下载目标平台的 sysroot（debian/ubuntu/alpine）        │     │
│  │  - 管理依赖的 C 库（openssl/pcap/sqlite 的交叉编译版本）    │     │
│  │  - 生成链接器脚本                                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    LinkerConfig                              │     │
│  │  - 自动配置目标平台的链接器                                  │     │
│  │  - 支持：gcc / lld / mold                                  │     │
│  │  - 生成 .cargo/config.toml                                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompatibilityDB                          │     │
│  │  - 已知兼容的 crate 列表                                    │     │
│  │  - 交叉编译常见问题数据库                                    │     │
│  │  - 自动检测潜在的交叉编译问题                                │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心类型定义

```rust
// crates/bt-toolchain/src/lib.rs

// ─── 目标平台管理 ───

/// 已安装的交叉编译目标
#[derive(Debug, Clone, Serialize)]
pub struct InstalledTarget {
    pub triple: TargetTriple,
    pub sysroot_path: PathBuf,
    pub linker: Option<PathBuf>,
    pub std_available: bool,
    pub installed_at: DateTime<Utc>,
}

/// 工具链管理器
pub struct ToolchainManager {
    /// 目标存储路径
    install_dir: PathBuf,
    /// 已安装的目标列表
    installed: HashMap<String, InstalledTarget>,
    /// 系统已安装的链接器
    system_linkers: Vec<String>,
}

impl ToolchainManager {
    /// 检测当前系统上已安装的 all 目标
    pub fn detect_all() -> Result<Self, ToolchainError>;

    /// 安装目标平台
    pub async fn install(&mut self, target: &TargetTriple, options: &InstallOptions) -> Result<InstalledTarget>;

    /// 卸载目标平台
    pub async fn uninstall(&mut self, target: &TargetTriple) -> Result<()>;

    /// 列出已安装目标
    pub fn list_installed(&self) -> Vec<InstalledTarget>;

    /// 检查目标是否可用
    pub fn is_available(&self, target: &TargetTriple) -> bool;

    /// 获取链接器路径
    pub fn get_linker(&self, target: &TargetTriple) -> Result<PathBuf, ToolchainError>;
}

// ─── 安装选项 ───

/// 安装配置
#[derive(Debug, Clone)]
pub struct InstallOptions {
    /// Sysroot 类型
    pub sysroot: SysrootType,
    /// 链接器类型
    pub linker: LinkerType,
    /// 是否同时安装 Rust std
    pub with_rustup_std: bool,
    /// C 库版本
    pub sysroot_version: Option<String>,
}

pub enum SysrootType {
    /// 使用 debian sysroot（默认）
    Debian,
    /// 使用 alpine sysroot（musl）
    Alpine,
    /// 使用 ubuntu sysroot
    Ubuntu,
    /// 自定义 sysroot 路径
    Custom(PathBuf),
}

pub enum LinkerType {
    /// 自动选择（优先 mold > lld > gcc）
    Auto,
    /// 指定链接器名称
    Named(String),
}

// ─── Sysroot 提供商 ───

/// Sysroot 自动下载
pub struct SysrootProvider;

impl SysrootProvider {
    /// 下载并解压 sysroot
    pub async fn download_sysroot(
        target: &TargetTriple,
        sysroot_type: SysrootType,
        dest: &Path,
    ) -> Result<PathBuf, ToolchainError>;

    /// 获取预编译 sysroot 下载地址
    fn get_download_url(target: &TargetTriple, sysroot_type: SysrootType) -> Result<String>;
}

// ─── 链接器配置 ───

/// 链接器配置生成器
pub struct LinkerConfig;

impl LinkerConfig {
    /// 检测系统上可用的链接器
    pub fn detect_system_linkers() -> Vec<String>;

    /// 生成链接器包装脚本
    pub fn generate_linker_wrapper(
        target: &TargetTriple,
        linker: &str,
        sysroot: &Path,
    ) -> Result<String>;

    /// 生成 .cargo/config.toml 片段
    pub fn generate_cargo_config(
        target: &TargetTriple,
        linker: &Path,
    ) -> String;
}
```

### 2.3 CLI 交互

```
$ bt target list
Installed targets:
  x86_64-unknown-linux-gnu        (host, native)
  aarch64-unknown-linux-gnu       ✓ sysroot, ✓ linker (aarch64-linux-gnu-gcc)

$ bt target add aarch64-unknown-linux-gnu
  ✓ Installing sysroot: debian-bookworm (aarch64)
  ✓ Installing linker: aarch64-linux-gnu-gcc
  ✓ Installing Rust std (via rustup)
  ✓ Generating .cargo/config.toml
  ───────────────────────────────────
  ✅ aarch64-unknown-linux-gnu 安装完成

$ bt build --target aarch64-unknown-linux-gnu
  ✓ Compiling for aarch64-unknown-linux-gnu
  ✓ Using linker: aarch64-linux-gnu-gcc
  ✓ Building main.rs → target/aarch64-unknown-linux-gnu/release/hello
```

### 2.4 与 bt-std-bridge 的集成

```rust
// bt-std-bridge 扩展

impl StdBridge {
    /// 获取交叉编译的标准库路径
    pub fn cross_std_lib_paths(
        &self,
        target: &TargetTriple,
        toolchain: &ToolchainManager,
    ) -> Result<Vec<PathBuf>, StdError> {
        if target.is_host() {
            return self.std_lib_paths(target);
        }

        let installed = toolchain.get_installed(target)?;
        Ok(vec![
            installed.sysroot_path.join("lib/rustlib").join(target.to_string()).join("lib"),
        ])
    }
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **目标平台管理** | `bt target add/remove/list` |
| C2 | **Sysroot 自动下载** | 支持 Debian/Alpine/Ubuntu sysroot |
| C3 | **链接器自动配置** | 自动检测和配置交叉链接器 |
| C4 | **C 库依赖管理** | 自动安装 openssl/pcap 等 C 库的交叉编译版本 |
| C5 | **cargo 配置生成** | 自动生成 `.cargo/config.toml` |
| C6 | **兼容性数据库** | 已知兼容/不兼容的 crate 列表 |
| C7 | **Docker 集成** | 可选基于 Docker 的交叉编译环境 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-toolchain` | 工具链管理 + Sysroot 下载 + 链接器配置 | ~1,500 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-std-bridge` | 新增 `cross_std_lib_paths()` | 低 |
| `bt-cli` | 新增 `target` 子命令 | 低 |

### CLI 变更

```bash
# ─── 目标管理 ───

bt target list                           # 列出已安装目标
bt target add <triple>                   # 安装交叉编译目标
bt target remove <triple>                # 卸载
bt target update <triple>                # 更新

bt target add aarch64-unknown-linux-gnu
bt target add x86_64-unknown-linux-musl  # 静态链接 musl
bt target add wasm32-unknown-unknown      # WASM 目标

# ─── 交叉编译 ───

bt build --target aarch64-unknown-linux-gnu
bt build --target wasm32-unknown-unknown

# ─── Docker 交叉编译 ───

bt target docker <triple>                # 使用 Docker 环境编译
```

---

## 5. 分阶段实施

### Phase 3 扩展（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-toolchain` crate: 目标管理 + rustup 集成 | `bt target list` + `bt target add aarch64` |

### Phase 4 扩展（1 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 2 | Sysroot 自动下载 + 链接器配置 + cargo config | `bt build --target` 完整可用 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| Sysroot 下载源不可用 | 中 | 中 | 多个镜像源 + 缓存 |
| 链接器兼容性问题 | 中 | 中 | 支持 gcc/lld/mold 多种选择 |
| C 库依赖解析复杂 | 高 | 中 | 首版仅支持无 C 依赖的纯 Rust 项目 |
