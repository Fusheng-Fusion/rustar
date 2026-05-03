# F12 — 跨语言编译时反射（Cross-Language Compile-Time Reflection）

> **优先级**：🟢 低 | **复杂度**：高 | **建议 Phase**：Phase 5+ | **工期**：~6 周

---

## 1. 概述

BlockType 已经通过 BTIR 统一了 Rust 和 TypeScript 的中间表示。但 IR 层只用于编译——一旦编译完成，类型信息就丢失了。本方案在 BTIR 层定义**标准反射元数据格式**，使得编译后的产物可以保留完整的类型结构信息，实现**跨语言的编译时反射**：Rust 代码可以查询 TypeScript 类型的 IR 元数据，反之亦然。

### 核心理念

```
场景 1：Rust 调用 TypeScript 代码
  #[bt_import(module = "./utils.ts")]
  fn greet(name: String) -> String;
  // 编译时自动从 TypeScript 的 IR 元数据中获取函数签名和类型信息

场景 2：跨语言序列化
  #[derive(BtSerialize)]
  struct User { name: String, age: u32 }
  // 自动为 TypeScript 生成对应的类型声明和序列化/反序列化代码

场景 3：编译时类型查询
  // Rust 代码中：
  bt_reflect::type_info::<User>()  // 返回完整的类型结构（字段名、类型、偏移量等）
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     bt-reflection crate                             │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    ReflectionMetadata                        │     │
│  │  - 标准编译时反射元数据格式                                │     │
│  │  - 存储在 IRModule 中作为一等公民                         │     │
│  │  - 序列化为 JSON/bincode 格式                              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    SchemaGenerator                           │     │
│  │  - 从 IR 类型生成跨语言类型声明                           │     │
│  │  - TypeScript 接口声明 (.d.ts)                            │     │
│  │  - Rust 类型定义                                          │     │
│  │  - JSON Schema                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    SerializationBridge                       │     │
│  │  - 跨语言序列化架构自动生成                                │     │
│  │  - 支持：JSON / bincode / MessagePack                     │     │
│  │  - 编译时绑定检查（Rust 结构体字段变更 → TypeScript 告警） │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompileTimeQueryAPI                       │     │
│  │  - 编译时可调用的类型查询 API                              │     │
│  │  - type_info / field_info / method_info / trait_info        │     │
│  │  - 宏或 proc-macro 调用入口                               │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心类型定义

```rust
// crates/bt-reflection/src/lib.rs

// ─── 反射元数据 ───

/// 编译时反射元数据（存储在 IRModule 中）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReflectionMetadata {
    /// 模块中所有类型的反射信息
    pub types: Vec<TypeReflection>,
    /// 模块中所有函数签名的反射信息
    pub functions: Vec<FunctionReflection>,
    /// 模块中所有 trait/interface 的反射信息
    pub traits: Vec<TraitReflection>,
    /// 语言来源
    pub source_language: String,  // "rust" / "typescript"
}

/// 类型的编译时反射信息
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TypeReflection {
    pub name: String,
    pub kind: ReflectionTypeKind,
    pub fields: Vec<FieldReflection>,
    pub methods: Vec<MethodSignature>,
    pub generic_params: Vec<String>,
    pub implements: Vec<String>,        // 实现的 trait/interface
    pub metadata: HashMap<String, String>,  // 自定义属性（如 serde rename）
}

pub enum ReflectionTypeKind {
    Struct,
    Enum,
    Union,           // TypeScript union type
    Interface,       // TypeScript interface
    TypeAlias,
}

/// 字段反射信息
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldReflection {
    pub name: String,
    pub type_name: String,             // 类型名称（跨语言唯一标识）
    pub type_ref: TypeReflectionId,    // 指向类型定义
    pub offset: Option<u64>,           // 内存偏移量（布局已知时）
    pub attributes: Vec<String>,       // 属性/装饰器
}

/// 函数签名反射
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FunctionReflection {
    pub name: String,
    pub params: Vec<ParamReflection>,
    pub return_type: String,
    pub is_async: bool,
    pub source_language: String,
}

pub struct ParamReflection {
    pub name: String,
    pub type_name: String,
    pub optional: bool,               // TypeScript optional param
    pub default_value: Option<String>,
}

// ─── 跨语言类型映射 ───

/// 跨语言类型映射表
pub struct CrossLanguageTypeMapper {
    /// Rust 类型 → BTIR 类型 → TypeScript 类型 的映射
    map: HashMap<String, TypeMapping>,
}

pub struct TypeMapping {
    pub rust_type: String,
    pub btir_type: IRType,
    pub ts_type: String,
    pub json_schema: String,
}

impl CrossLanguageTypeMapper {
    pub fn builtin_mappings() -> Self;

    /// 将 BTIR 类型映射为 TypeScript 类型
    pub fn to_typescript(&self, btir_type: &IRType) -> String;

    /// 将 BTIR 类型映射为 JSON Schema
    pub fn to_json_schema(&self, btir_type: &IRType) -> serde_json::Value;

    /// 注册自定义类型映射
    pub fn register_mapping(&mut self, rust: &str, ts: &str, json_schema: &str);
}

// ─── Schema 生成器 ───

/// Schema 生成器 — 从 IR 反射元数据生成跨语言类型声明
pub struct SchemaGenerator {
    type_mapper: CrossLanguageTypeMapper,
}

impl SchemaGenerator {
    /// 生成 TypeScript 接口声明 (.d.ts)
    pub fn generate_typescript_decls(
        &self,
        metadata: &ReflectionMetadata,
    ) -> String;

    /// 生成 Rust 类型定义
    pub fn generate_rust_types(
        &self,
        metadata: &ReflectionMetadata,
    ) -> String;

    /// 生成 JSON Schema
    pub fn generate_json_schema(
        &self,
        metadata: &ReflectionMetadata,
    ) -> serde_json::Value;

    /// 生成序列化/反序列化代码
    pub fn generate_serialization(
        &self,
        metadata: &ReflectionMetadata,
        format: SerializationFormat,
    ) -> SerializationCode;
}

// ─── 编译时查询 API ───

/// 编译时可调用的类型查询
///
/// 通过 proc-macro 调用：
///   let info = bt_reflect::type_info::<MyStruct>();
///   let fields = bt_reflect::fields::<MyStruct>();
///
/// 底层实现：
///   - 编译时从 IRModule 的 ReflectionMetadata 中查找
///   - 返回的信息在编译期完全确定
pub mod bt_reflect {
    /// 获取类型的反射信息
    pub fn type_info<T: Reflectable>() -> TypeReflection;

    /// 获取类型的字段列表
    pub fn fields<T: Reflectable>() -> Vec<FieldReflection>;

    /// 检查类型是否实现某个 trait
    pub fn implements<T: Reflectable>(trait_name: &str) -> bool;

    /// 跨语言类型查询（从 TypeScript IR 元数据查询）
    pub fn cross_language_type_info(
        language: &str,
        type_name: &str,
    ) -> Result<TypeReflection, ReflectError>;
}
```

### 2.3 与 IR 的集成

```rust
// IRModule 扩展反射元数据

// 现有 IRModule 结构（02-Core-Types.md）增加：

pub struct IRModule {
    // ... 现有字段 ...

    /// 编译时反射元数据（新增）
    pub reflection: Option<ReflectionMetadata>,
}

// 在 Lower 阶段提取反射信息：
impl MirToBtirConverter {
    /// IR 转换时同步提取反射元数据
    fn extract_reflection(
        &self,
        tcx: TyCtxt<'_>,
        ir_module: &mut IRModule,
    ) {
        let metadata = ReflectionMetadata {
            types: self.extract_types(tcx),
            functions: self.extract_functions(tcx),
            traits: self.extract_traits(tcx),
            source_language: "rust".into(),
        };
        ir_module.reflection = Some(metadata);
    }
}
```

### 2.4 跨语言反射工作流

```
Rust 代码:                           TypeScript 代码:
  #[derive(BtSerialize)]                // user.ts
  struct User {                          interface User {
    name: String,                          name: string;
    age: u32,                              age: number;
  }                                      }

     │                                      │
     ▼                                      ▼
  bt-rustc-bridge                       bt-ts2rs（ts2rs 转译）
     │                                      │
     ▼                                      ▼
  BTIR (rust dialect)                   Rust 源码 → bt-rustc-bridge
  + ReflectionMetadata                  + ReflectionMetadata（含 bt_ts 标注）
     │                                      │
     ▼                                      ▼
  ┌─────────────────────────────────────────────┐
  │            BlockType IR 融合层                 │
  │                                              │
  │  1. 两种 IR 的 ReflectionMetadata 合并       │
  │  2. 跨语言类型映射解析                        │
  │  3. 生成跨语言类型声明 (.d.ts / .rs)         │
  │  4. 生成序列化/反序列化桥梁代码                │
  └─────────────────────────────────────────────┘
     │
     ▼
  输出：
    - .d.ts 声明文件（供 TypeScript 使用）
    - rs 桥接代码（供 Rust 使用）
    - JSON Schema（供 API 文档使用）
    - 序列化桥梁代码（运行时）
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **标准反射元数据** | 类型/函数/trait 的结构化元数据格式 |
| C2 | **跨语言类型映射** | Rust ↔ TypeScript 类型自动映射 |
| C3 | **Schema 生成** | 自动生成 .d.ts / JSON Schema |
| C4 | **序列化桥梁** | 跨语言序列化/反序列化代码自动生成 |
| C5 | **编译时类型查询** | 编译期反射 API（非运行时） |
| C6 | **跨语言类型检查** | Rust 结构体字段变更 → TypeScript 侧告警 |
| C7 | **跨语言方法调用** | Rust 中调用 TypeScript 函数（编译时绑定） |
| C8 | **属性/装饰器透传** | Rust 的 `#[serde(rename)]` → TypeScript 的 `@JsonProperty` |

---

## 4. 与现有架构的集成

### 新增 crate

| Crate | 职责 | 行数估算 |
|-------|------|---------|
| `bt-reflection` | 反射元数据 + Schema 生成 + 类型映射 + 查询 API | ~3,000 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-ir` | `IRModule` 增加 `reflection: Option<ReflectionMetadata>` | 低 |
| `bt-rustc-bridge` | `MirToBtirConverter` 提取反射信息 | 中 |
| `bt-ts2rs` | 从 TS→Rust 转译过程提取反射信息 + bt_ts 标注 | 中 |

---

## 5. 分阶段实施

### Phase 5 扩展（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | `bt-reflection` crate: ReflectionMetadata + TypeReflection | 反射元数据格式可用 |
| 2 | IRModule 反射集成 + Rust 侧数据提取 | `bt reflect` 显示类型信息 |
| 3 | SchemaGenerator: TypeScript 声明生成 + JSON Schema | `bt reflect --ts-decls` |

### 扩展阶段（3 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 4 | 跨语言类型映射 + TypeScript 侧提取 | 双向反射可用 |
| 5 | 序列化桥梁生成 + 运行时库 | 跨语言序列化可用 |
| 6 | 编译时类型查询 API + proc-macro | `bt_reflect::type_info!()` 可用 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 反射元数据增加 IR 体积 | 中 | 低 | 可选特性，`--reflection` 参数控制 |
| 跨语言类型映射精确度有限 | 中 | 中 | 内置 50+ 通用类型映射 + 自定义映射注册 |
| Rust 和 TypeScript 类型系统差异 | 高 | 中 | 仅映射可表达的共同子集，差异部分标注 |
| 反射信息在优化 Pass 中过期 | 中 | 低 | Pass 执行后自动更新反射元数据 |
