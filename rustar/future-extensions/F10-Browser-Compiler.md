# F10 — 浏览器内编译器（WASM 化）

> **优先级**：🟢 低 | **复杂度**：高 | **建议 Phase**：Phase 5+ | **工期**：~8 周

---

## 1. 概述

当前 BlockType 作为 Rust 可执行文件运行，依赖 `rustc_driver`（sysroot）。但 BTIR 核心（`bt-core` + `bt-ir` + `bt-ir-verifier` + `bt-passes`）**不依赖任何 rustc crate**，可以独立编译为 WASM。本方案将 BlockType 的非 rustc 依赖部分编译为 WASM，使其可以在浏览器中直接运行——用于在线 IDE 代码分析、Lint 预览、AI 建议展示等场景。

### 核心理念

```
编译器 WASM 化 ≠ 把整个编译器搬到浏览器
  而是：把不依赖 rustc 的核心 IR 层 + Pass 引擎 + AI 编排器搬到浏览器
  rustc 前端部分仍然在服务器端

浏览器内（WASM）：
  - bt-core（基础类型）
  - bt-ir（IR 构建/操作/序列化）
  - bt-ir-verifier（IR 验证）
  - bt-passes（优化 Pass，Dialect 降级 Pass）
  - bt-ai（AI 编排器，仅规则引擎部分）

服务器端（Native）：
  - bt-rustc-bridge（rustc 前端对接）
  - bt-cargo（Cargo 解析）
  - bt-backend-llvm（LLVM 代码生成）
  - bt-backend-cranelift（Cranelift 代码生成）
```

---

## 2. 架构设计

### 2.1 整体架构

```
浏览器 (WASM)                          服务器 (Native)
─────────────                          ────────────────
┌──────────────────┐                   ┌──────────────────┐
│  Online IDE       │                   │  BlockType Server │
│  (VSCode.dev)     │                   │  (axum)           │
│                   │  REST / WebSocket │                   │
│  ┌────────────┐   │ ◀──────────────▶ │  ┌────────────┐   │
│  │ bt-ir-wasm │   │  IR JSON / Diag  │  │bt-rustc    │   │
│  │ (WASM)     │   │                  │  │-bridge     │   │
│  │            │   │                  │  └────────────┘   │
│  │ - IR 操作  │   │                  │                   │
│  │ - Pass 运行│   │                  │  ┌────────────┐   │
│  │ - IR 验证  │   │                  │  │bt-backend  │   │
│  │ - AI 规则  │   │                  │  │-llvm       │   │
│  └────────────┘   │                  │  └────────────┘   │
└──────────────────┘                   └──────────────────┘

工作流：
  1. 用户在浏览器中编写代码
  2. 源码发送到服务器 → bt-rustc-bridge 编译 → 返回 BTIR (JSON)
  3. BTIR 缓存在浏览器端（WASM 侧）
  4. 用户修改代码 → 浏览器中的 bt-ir-wasm 做快速增量检查
  5. 需要完整编译时 → 发送到服务器
```

### 2.2 WASM crate 结构

```toml
# crates/bt-ir-wasm/Cargo.toml

[package]
name = "bt-ir-wasm"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
# BTIR 核心（无 rustc 依赖）
bt-ir = { path = "../bt-ir" }
bt-ir-verifier = { path = "../bt-ir-verifier" }
bt-passes = { path = "../bt-passes" }
bt-dialect-core = { path = "../bt-dialect-core" }
bt-ai = { path = "../bt-ai", features = ["rule-engine-only"] }
bt-core = { path = "../bt-core" }
bt-event-store = { path = "../bt-event-store" }

# WASM 绑定
wasm-bindgen = "0.2"
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
wasm-bindgen-futures = "0.4"
```

### 2.3 WASM API 定义

```rust
// crates/bt-ir-wasm/src/lib.rs

use wasm_bindgen::prelude::*;

/// 浏览器中的 BlockType IR 引擎
#[wasm_bindgen]
pub struct BtIrEngine {
    /// IR 模块缓存
    modules: Vec<IRModule>,
    /// 验证器
    verifier: IRVerifier,
    /// Pass 管理器（仅 Dialect 降级和验证 Pass）
    pass_manager: PassManager,
    /// AI 规则引擎（本地运行）
    ai_engine: RuleBasedProvider,
}

#[wasm_bindgen]
impl BtIrEngine {
    /// 创建引擎实例
    #[wasm_bindgen(constructor)]
    pub fn new() -> Result<BtIrEngine, JsValue>;

    /// 加载 IR 模块（从服务器返回的 JSON）
    pub fn load_ir_module(&mut self, json: &str) -> Result<String, JsValue> {
        let module: IRModule = serde_json::from_str(json)
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        let id = module.id.to_string();
        self.modules.push(module);
        Ok(id)
    }

    /// 运行 IR 验证
    pub fn verify_ir(&self, module_id: &str) -> Result<JsValue, JsValue> {
        let module = self.find_module(module_id)?;
        let result = self.verifier.verify(module);
        serde_wasm_bindgen::to_value(&result)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// 运行 Dialect 降级 Pass
    pub fn lower_dialects(&mut self, module_id: &str) -> Result<String, JsValue> {
        let module = self.find_module_mut(module_id)?;
        self.pass_manager.run_pass(module, "dialect_lower")
            .map_err(|e| JsValue::from_str(&e.to_string()))?;
        serde_json::to_string(module)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// AI 快速分析（规则引擎，本地运行）
    pub fn ai_analyze(&self, module_id: &str) -> Result<JsValue, JsValue> {
        let module = self.find_module(module_id)?;
        let suggestions = self.ai_engine.analyze_sync(module);
        serde_wasm_bindgen::to_value(&suggestions)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// 导出 IR 为 JSON
    pub fn export_ir_json(&self, module_id: &str) -> Result<String, JsValue> {
        let module = self.find_module(module_id)?;
        serde_json::to_string_pretty(module)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }

    /// 比较两个 IR 的差异（用于增量结果展示）
    pub fn diff_ir(&self, old_id: &str, new_id: &str) -> Result<JsValue, JsValue> {
        let old = self.find_module(old_id)?;
        let new = self.find_module(new_id)?;
        let diff = compute_ir_diff(old, new);
        serde_wasm_bindgen::to_value(&diff)
            .map_err(|e| JsValue::from_str(&e.to_string()))
    }
}
```

### 2.4 浏览器端使用示例

```typescript
// 在线 IDE 中的使用

import init, { BtIrEngine } from 'bt-ir-wasm';

async function setupCompiler() {
    await init();  // 初始化 WASM

    const engine = new BtIrEngine();

    // 从服务器获取 BTIR（源码 → rustc 前端编译）
    const response = await fetch('/api/v1/compile', {
        method: 'POST',
        body: JSON.stringify({ source: code }),
    });
    const irJson = await response.text();
    const moduleId = engine.load_ir_module(irJson);

    // 浏览器端快速验证
    const verifyResult = engine.verify_ir(moduleId);
    showValidationErrors(verifyResult);

    // 浏览器端 AI 快速分析
    const aiSuggestions = engine.ai_analyze(moduleId);
    showAiSuggestions(aiSuggestions);

    // 用户修改代码后，浏览器端快速检查
    // 无需重新请求服务器
    const oldModuleId = moduleId;
    const newResponse = await fetch('/api/v1/compile', {
        method: 'POST',
        body: JSON.stringify({ source: modifiedCode }),
    });
    const newIrJson = await newResponse.text();
    const newModuleId = engine.load_ir_module(newIrJson);

    // 浏览器端 IR 差异对比
    const diff = engine.diff_ir(oldModuleId, newModuleId);
    showCodeChanges(diff);
}
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **IR 操作 WASM** | BTIR 核心类型可在浏览器中构建、修改、序列化 |
| C2 | **浏览器内 IR 验证** | 无需服务器即可验证 IR 完整性 |
| C3 | **浏览器内 Dialect 降级** | Dialect → bt_core 降级可在浏览器侧完成 |
| C4 | **浏览器内 AI 规则引擎** | 规则引擎（非 LLM）可在浏览器侧运行 |
| C5 | **IR 差异对比** | 浏览器内计算两次编译的 IR 差异 |
| C6 | **IR 可视化** | 基于 D3.js 在浏览器中渲染 IR 控制流图 |
| C7 | **离线分析** | 缓存的 IR 可以在离线时继续分析 |
| C8 | **增量客户端分析** | 修改代码后，浏览器侧快速做差异检查 |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-ir-wasm` | WASM 绑定 + 浏览器 API | ~1,500 |

### 非 WASM 化部分（保持不变）

- `bt-rustc-bridge` — 依赖 sysroot，无法 WASM 化
- `bt-backend-llvm` — 依赖 inkwell/LLVM 原生库
- `bt-backend-cranelift` — 支持 WASM 输出，但 Cranelift 本身需要 native

---

## 5. 分阶段实施

### Phase 5 扩展（4 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-ir-wasm` crate 搭建 + bt-core/bt-ir WASM 编译 | 基础 IR 类型可在 WASM 中使用 |
| 2 | bt-ir-verifier + bt-passes WASM 化 | IR 验证和 Pass 可在浏览器运行 |
| 3 | bt-ai 规则引擎 WASM 化（仅 rule-engine-only 模式） | AI 规则分析在浏览器端运行 |
| 4 | TypeScript 绑定 + NPM 包发布 | `@blocktype/ir-wasm` 可在 Node/浏览器中使用 |

### 扩展阶段（4 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 5-6 | HTTP API 客户端封装（浏览器主动请求服务器编译） | 可搭建完整工作流 |
| 7-8 | IR 可视化 + IR 差异对比 UI + VSCode.dev 集成 | 在线 IDE 可用 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| WASM 包体积过大 | 高 | 中 | 按需加载 + tree shaking + 压缩 |
| 浏览器中 LLM AI 不可用 | 中 | 低 | 规则引擎替代，LLM 由服务器提供 |
| rustc 驱动不可 WASM 化 | 确定 | 中 | 仅非 rustc 部分 WASM 化，rustc 依赖走服务器 |
| wasm-bindgen 序列化性能 | 中 | 中 | 大块数据用 JSON（字符串传输）代替 serde-wasm-bindgen |
