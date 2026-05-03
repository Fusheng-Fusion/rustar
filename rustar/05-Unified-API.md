# 05 — 统一 RESTful API 设计

## 核心思想

编译器的所有功能（编译、查询、监控、AI 分析）都通过 HTTP REST 端点暴露，**按关注点分层版本控制**。所有 API handler 内部调用同一个 `tower::Service` 实例——无论是 HTTP 外部调用还是内存内部调用，接口完全一致。

## 2.1 API 分层设计

| 分层 | 路径前缀 | 变更频率 | 说明 |
|------|---------|---------|------|
| 编译操作 | `/api/v1/compile/*` | 高频 | 编译管线各阶段 |
| IR 操作 | `/api/v1/ir/*` | 中频 | IR 构造/查询/验证 |
| 管理操作 | `/api/v1/admin/*` | 低频 | 前端/后端/插件注册表 |
| AI 操作 | `/api/v1/ai/*` | 独立版本 | AI 分析/对话/编排 |
| 监控操作 | `/api/v1/telemetry/*` | 低频 | 可观测性查询 |
| WebSocket | `/ws/v1/*` | 独立版本 | 实时推送 |
| LSP | stdin/stdout | 独立版本 | IDE 集成 |

---

## 2.2 编译操作 API (`/api/v1/compile/*`)

```
POST   /api/v1/compile                       # 提交完整编译任务
       Body: { source, frontend, backend: "llvm"|"cranelift"|"rustc"|"auto", options }
       Response: { task_id, status: "queued" }

POST   /api/v1/compile/lex                   # 仅词法分析
POST   /api/v1/compile/parse                 # 仅语法分析
POST   /api/v1/compile/analyze               # 仅语义分析
POST   /api/v1/compile/lower                 # 仅 IR 降低
POST   /api/v1/compile/optimize              # 仅 IR 优化
POST   /api/v1/compile/codegen               # 仅代码生成

GET    /api/v1/compile/tasks/{task_id}       # 查询任务状态
GET    /api/v1/compile/tasks/{task_id}/result          # 获取编译结果
GET    /api/v1/compile/tasks/{task_id}/diagnostics     # 获取诊断
GET    /api/v1/compile/tasks/{task_id}/artifacts/{stage} # 获取中间产物
DELETE /api/v1/compile/tasks/{task_id}       # 取消任务
```

## 2.3 IR 操作 API (`/api/v1/ir/*`)

```
POST   /api/v1/ir/load                       # 加载 IR 模块
GET    /api/v1/ir/modules/{id}               # IR 模块详情
GET    /api/v1/ir/modules/{id}/functions     # 列出函数
GET    /api/v1/ir/modules/{id}/types         # 列出类型
GET    /api/v1/ir/modules/{id}/dialects      # 列出已注册 Dialect
POST   /api/v1/ir/verify                     # 验证 IR
POST   /api/v1/ir/optimize                   # 运行优化 Pass
POST   /api/v1/ir/diff                       # IR 差异比较（文本/结构化/可视）
GET    /api/v1/ir/dialects                   # 列出所有已注册 Dialect
GET    /api/v1/ir/dialects/{name}            # Dialect 详情（操作定义表）
```

### 2.3.1 IR Diff API — 详细 Schema

`POST /api/v1/ir/diff` 支持三种 diff 格式输出，用于 Dashboard 可视化和 AI 变更分析。

**请求体**：

```json
{
  "before": {
    "source": "string (源代码) | ir_module_id (已有 IR 模块 ID)",
    "type": "source | ir_module"
  },
  "after": {
    "source": "string (源代码) | ir_module_id (已有 IR 模块 ID)",
    "type": "source | ir_module"
  },
  "format": "text | structured | visual",
  "frontend": "typescript",
  "options": {
    "context_lines": 3,
    "include_metadata": true,
    "dialect_filter": ["bt_ts"]
  }
}
```

**响应体**（按 `format` 分三种）：

```jsonc
// format = "text" → 统一 diff 文本
{
  "format": "text",
  "diff": "--- a/app.ts\n+++ b/app.ts\n@@ -1,3 +1,4 @@\n function add(a, b) {\n-  return a + b;\n+  return a + b + 1;\n }\n",
  "stats": { "added": 1, "removed": 1, "changed": 1 }
}

// format = "structured" → 结构化 diff（按 IR 节点）
{
  "format": "structured",
  "changes": [
    {
      "kind": "function_modified",
      "node_id": "fn_add_001",
      "name": "add",
      "before_summary": "params: [a: number, b: number] → number",
      "after_summary": "params: [a: number, b: number] → number",
      "body_diff": { "added_ops": 1, "removed_ops": 1, "changed_ops": 0 },
      "children": [
        {
          "kind": "operation_changed",
          "node_id": "op_ret_001",
          "before": "Return(BinaryOp(Add, Load(a), Load(b)))",
          "after": "Return(BinaryOp(Add, BinaryOp(Add, Load(a), Load(b)), Const(1)))"
        }
      ]
    }
  ],
  "stats": { "functions_added": 0, "functions_removed": 0, "functions_modified": 1, "types_changed": 0 }
}

// format = "visual" → Dashboard 可视化数据
{
  "format": "visual",
  "graph_delta": {
    "nodes_added": [],
    "nodes_removed": [],
    "nodes_modified": [
      { "id": "fn_add_001", "label": "add()", "change_type": "modified", "impact": "low" }
    ],
    "edges_added": [],
    "edges_removed": []
  },
  "highlights": [
    { "range": { "start_line": 2, "end_line": 2 }, "type": "modification", "severity": "info" }
  ]
}
```

## 2.4 管理操作 API (`/api/v1/admin/*`)

```
GET    /api/v1/admin/frontends               # 列出已注册前端
GET    /api/v1/admin/frontends/{name}        # 前端详情
GET    /api/v1/admin/frontends/{name}/capabilities

GET    /api/v1/admin/backends                # 列出已注册后端
GET    /api/v1/admin/backends/{name}         # 后端详情
GET    /api/v1/admin/backends/{name}/targets # 支持的目标平台

GET    /api/v1/admin/plugins                 # 已加载插件
POST   /api/v1/admin/plugins/load            # 加载 WASM 插件
DELETE /api/v1/admin/plugins/{name}          # 卸载插件
GET    /api/v1/admin/plugins/{name}/status   # 插件状态（沙箱内存/CPU 使用）

GET    /api/v1/admin/status                  # 系统状态总览
```

## 2.5 AI 操作 API (`/api/v1/ai/*`)

```
POST   /api/v1/ai/analyze                    # AI 代码分析（支持 SSE 流式）
POST   /api/v1/ai/optimize                   # AI 优化建议（支持 SSE 流式）
POST   /api/v1/ai/explain                    # AI 解释编译过程
POST   /api/v1/ai/chat                       # AI 对话（上下文感知）
GET    /api/v1/ai/providers                  # 可用 AI 提供商 + 健康状态
POST   /api/v1/ai/providers/{name}/config    # 配置提供商
GET    /api/v1/ai/budget                     # 当前 token 预算使用情况
GET    /api/v1/ai/cache/stats                # AI 结果缓存统计
```

## 2.6 可观测性 API (`/api/v1/telemetry/*`)

```
GET    /api/v1/telemetry/pipeline            # 管线实时状态
GET    /api/v1/telemetry/traces/{task_id}    # 编译 Trace
GET    /api/v1/telemetry/metrics             # 性能指标
GET    /api/v1/telemetry/logs                # 编译日志流
GET    /api/v1/telemetry/events              # 事件溯源查询（支持分页）
GET    /api/v1/telemetry/events/{task_id}    # 按任务回放事件历史
GET    /api/v1/telemetry/dependency-graph/{task_id} # 依赖图导出
```

## 2.7 WebSocket (`/ws/v1/*`)

```
WS     /ws/v1/pipeline                       # 管线实时推送
WS     /ws/v1/diagnostics                   # 诊断实时推送
WS     /ws/v1/progress                      # 编译进度
```

---

## 2.8 统一响应格式

```json
{
  "request_id": "uuid-v4",
  "timestamp": "2026-05-02T04:00:00Z",
  "trace_id": "otel-trace-id",
  "span_id": "otel-span-id",
  "status": "success | error | partial",
  "data": { },
  "diagnostics": [
    {
      "severity": "error | warning | info | note",
      "code": "E0001",
      "message": "...",
      "location": { "file": "main.rs", "line": 10, "col": 5 },
      "suggestions": [ ]
    }
  ],
  "metrics": {
    "duration_ms": 42,
    "memory_peak_kb": 1024,
    "phases": {
      "lex": { "duration_ms": 2, "items": 150 },
      "parse": { "duration_ms": 8, "items": 45 },
      "analyze": { "duration_ms": 12, "items": 30 },
      "lower": { "duration_ms": 5, "items": 25 },
      "optimize": { "duration_ms": 10, "items": 25 },
      "codegen": { "duration_ms": 5, "items": 1 }
    }
  },
  "pagination": {
    "cursor": "eyJpZCI6MX0=",
    "has_more": true,
    "total_count": 1500
  }
}
```

## 2.9 错误码体系

| 范围 | 领域 | 示例 |
|------|------|------|
| `E0xxx` | 前端错误（词法/语法/语义） | `E0001` 未闭合字符串, `E0101` 类型不匹配 |
| `E1xxx` | IR 错误（验证/序列化） | `E1001` IR 验证失败, `E1002` Dialect 未注册 |
| `E2xxx` | 后端错误（代码生成/链接） | `E2001` 目标平台不支持, `E2002` 链接失败 |
| `E3xxx` | 系统错误（服务/缓存/插件） | `E3001` 服务超时, `E3002` 插件加载失败 |
| `E4xxx` | AI 错误（provider/预算/超时） | `E4001` provider 不可用, `E4002` token 预算超限 |

---

## 2.10 认证与多租户

### 2.10.1 设计原则

| 原则 | 说明 |
|------|------|
| **本地优先** | 单用户本地模式无需认证，开箱即用 |
| **渐进式安全** | 服务化部署时按需启用认证层 |
| **项目隔离** | 多用户场景下编译缓存、IR 模块、AI 预算按项目隔离 |
| **零侵入** | 认证逻辑通过 tower middleware 注入，不侵入业务 handler |

### 2.10.2 认证模式

| 模式 | 适用场景 | 认证方式 |
|------|---------|---------|
| `none` | 本地 CLI / 单用户 | 无认证，所有请求直接放行 |
| `api_key` | 团队内部服务 | `Authorization: Bearer <api_key>` |
| `jwt` | 多租户 SaaS | JWT RS256/HS256，含 tenant_id + project_id claims |

认证模式通过配置文件或环境变量 `BT_AUTH_MODE` 选择，启动时确定，不支持运行时热切换。

### 2.10.3 API Key 模式

```rust
/// API Key 认证配置
pub struct ApiKeyAuthConfig {
    /// 有效 API Key 列表（SHA-256 哈希存储）
    pub keys: Vec<HashedApiKey>,
    /// 默认项目（单项目场景）
    pub default_project: Option<ProjectId>,
}

/// tower middleware：校验 Bearer token
pub struct ApiKeyMiddleware<S> {
    inner: S,
    config: Arc<ApiKeyAuthConfig>,
}
```

请求示例：

```http
POST /api/v1/compile
Authorization: Bearer bt_live_k8s_a1b2c3d4e5f6
Content-Type: application/json
```

校验失败返回 `401 Unauthorized`：

```json
{
  "status": "error",
  "diagnostics": [{
    "severity": "error",
    "code": "E3003",
    "message": "Invalid or missing API key"
  }]
}
```

### 2.10.4 JWT 多租户模式

```rust
/// JWT 认证配置
pub struct JwtAuthConfig {
    /// RS256 公钥 或 HS256 密钥
    pub verification_key: VerificationKey,
    /// 签发者（issuer）
    pub issuer: String,
    /// 受众（audience）
    pub audience: String,
    /// Token 过期容忍（秒）
    pub leeway_secs: u64,
}

/// JWT Claims 结构
pub struct BlockTypeClaims {
    /// 标准字段
    pub sub: String,          // 用户 ID
    pub iss: String,          // 签发者
    pub aud: String,          // 受众
    pub exp: i64,             // 过期时间
    pub iat: i64,             // 签发时间
    /// BlockType 扩展字段
    pub tenant_id: String,    // 租户 ID
    pub project_id: String,   // 项目 ID
    pub roles: Vec<String>,   // 角色 ["reader", "compiler", "admin"]
}
```

请求示例：

```http
POST /api/v1/compile
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Tenant-ID: tenant_acme
X-Project-ID: proj_frontend_ts
```

### 2.10.5 项目隔离

多租户模式下，所有数据按 `(tenant_id, project_id)` 二级隔离：

| 隔离维度 | 说明 |
|---------|------|
| **编译缓存** | Salsa 数据库按 project 分库，互不可见 |
| **IR 模块** | IR 存储命名空间：`{tenant_id}/{project_id}/ir/{module_id}` |
| **诊断历史** | 按 project 独立存储和查询 |
| **AI Token 预算** | 按 project 独立配额，见 §2.10.6 |
| **插件沙箱** | WASM 插件按 project 隔离实例 |

```rust
/// 项目上下文——由认证中间件注入到请求扩展中
pub struct ProjectContext {
    pub tenant_id: TenantId,
    pub project_id: ProjectId,
    pub roles: Vec<Role>,
    pub budget: TokenBudget,
}

/// 从 tower ServiceRequest 中提取
impl ProjectContext {
    pub fn from_request(req: &ServiceRequest) -> Result<&Self, AuthError> {
        req.extensions().get::<ProjectContext>()
            .ok_or(AuthError::MissingContext)
    }
}
```

### 2.10.6 Token Budget（AI 调用配额）

防止 AI 调用爆炸，按 project 设定预算上限：

```rust
/// AI Token 预算配置
pub struct TokenBudget {
    /// 单次请求上限
    pub per_request_limit: usize,        // 默认: 4096 tokens
    /// 每日总预算
    pub daily_budget: usize,             // 默认: 1_000_000 tokens
    /// 每小时预算
    pub hourly_budget: usize,            // 默认: 100_000 tokens
    /// 预算耗尽策略
    pub on_exhausted: ExhaustionPolicy,  // 默认: FallbackToRuleBased
}

pub enum ExhaustionPolicy {
    /// 返回错误，拒绝 AI 请求
    Reject,
    /// 降级为基于规则的静态分析（无 AI 开销）
    FallbackToRuleBased,
}
```

查询当前预算：

```http
GET /api/v1/ai/budget
→ {
  "daily": { "used": 45000, "limit": 1000000, "remaining": 955000 },
  "hourly": { "used": 3200, "limit": 100000, "remaining": 96800 },
  "per_request_limit": 4096
}
```

### 2.10.7 权限矩阵

| API 路径 | `reader` | `compiler` | `admin` |
|---------|----------|------------|---------|
| `GET /api/v1/compile/tasks/*` | ✅ | ✅ | ✅ |
| `POST /api/v1/compile/*` | ❌ | ✅ | ✅ |
| `GET /api/v1/ir/*` | ✅ | ✅ | ✅ |
| `POST /api/v1/ir/*` | ❌ | ✅ | ✅ |
| `GET /api/v1/admin/*` | ❌ | ❌ | ✅ |
| `POST /api/v1/admin/*` | ❌ | ❌ | ✅ |
| `POST /api/v1/ai/*` | ❌ | ✅ | ✅ |
| `GET /api/v1/telemetry/*` | ✅ | ✅ | ✅ |

权限不足返回 `403 Forbidden`：

```json
{
  "status": "error",
  "diagnostics": [{
    "severity": "error",
    "code": "E3004",
    "message": "Insufficient permissions: role 'reader' cannot POST /api/v1/compile"
  }]
}
```

### 2.10.8 错误码扩展

在 §2.9 错误码体系中新增 `E3xxx` 认证相关错误码：

| 错误码 | 含义 |
|--------|------|
| `E3003` | 认证失败（API Key 无效 / JWT 过期） |
| `E3004` | 权限不足（角色不匹配） |
| `E3005` | 项目不存在或无权访问 |
| `E3006` | Token 预算耗尽（降级为规则分析） |

---

## 2.11 TypeScript 前端 — ts2rs 转译协议

> **变更 v3.1**：TypeScript 前端不再通过 JSON-RPC 桥接自定义 Lexer/Parser/TypeChecker，
> 改为 **ts2rs 转译方案**：Deno 进程仅作为官方 `typescript` npm 包的编译器 API 宿主，
> 接收 TypeScript 源码，返回类型信息 + AST 摘要，供 Rust 侧的 bt-ts2rs 转译器生成 Rust 代码。

### 2.11.1 通信方式

Deno 进程通过 stdin/stdout JSON-RPC 2.0 与 bt-ts2rs 通信。
协议版本：`v1`。
与旧方案的区别：

| 维度 | 旧方案（bt-frontend-ts） | 新方案（bt-ts2rs） |
|------|------------------------|-------------------|
| TypeScript 解析 | 自建 Lexer/Parser | 复用 `ts.createSourceFile()` |
| 类型检查 | 自建 TypeChecker | 复用 `checker.getTypeAtLocation()` |
| 输出 | bt_ts IR（JSON） | 类型信息 + AST 摘要 |
| Rust 侧职责 | 管理进程池 + JSON-RPC 桥接 | 将 TS AST → Rust AST |
| 进程数 | 进程池（1-4） | 单进程（或按需创建） |
| 运行时 | 自建 ts-runtime | 走 Rust 完整编译链路 |

### 2.11.2 Method 列表

| Method | 方向 | 说明 | 请求参数 | 成功结果 |
|--------|------|------|---------|---------|
| `initialize` | R→T | 版本协商 | `{ protocolVersion, capabilities }` | `{ protocolVersion, capabilities }` |
| `transpile` | R→T | 编译 TS 源码，返回类型信息 + AST | `[TranspileParams]` | `TranspileResult` |
| `shutdown` | R→T | 优雅关闭 | `{}` | `{}` |

### 2.11.3 `transpile` Method

```json
// bt-ts2rs → Deno 请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "transpile",
  "params": {
    "source": "interface User { name: string; age: number; }\nfunction greet(u: User): string { return `Hello ${u.name}`; }",
    "fileName": "app.ts",
    "options": {
      "target": "es2022",
      "strict": true,
      "extractTypeInfo": true,
      "extractAstSummary": true
    }
  }
}

// Deno → bt-ts2rs 成功响应
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "typeInfo": {
      "symbols": [
        {
          "name": "User",
          "kind": "interface",
          "members": [
            { "name": "name", "type": "string", "optional": false },
            { "name": "age", "type": "number", "optional": false }
          ]
        },
        {
          "name": "greet",
          "kind": "function",
          "signature": {
            "params": [
              { "name": "u", "type": "User" }
            ],
            "returnType": "string",
            "async": false
          }
        }
      ]
    },
    "astSummary": {
      "declarations": [
        {
          "kind": "InterfaceDeclaration",
          "name": "User",
          "startLine": 1,
          "endLine": 4
        },
        {
          "kind": "FunctionDeclaration",
          "name": "greet",
          "startLine": 5,
          "endLine": 5
        }
      ]
    },
    "diagnostics": [],
    "dependencies": []
  }
}
```

> **注意**：bt-ts2rs 接收 `typeInfo` + `astSummary` 后，在 Rust 侧使用 `syn` crate 生成 Rust 源码。
> 生成的 Rust 代码通过 `bt-rustc-bridge` 走完整 Rust 编译链路。
> 所有不可映射的 TS 模式在转译阶段报错，提示开发者替换写法。

### 2.11.4 JSON-RPC 错误码

| 错误码 | 含义 | 说明 |
|--------|------|------|
| `-32700` | Parse error | JSON 解析失败 |
| `-32600` | Invalid Request | 请求格式不合法 |
| `-32601` | Method not found | 不支持的 method |
| `-32602` | Invalid params | 参数校验失败 |
| `-32603` | Internal error | TS 编译器内部错误 |
| `-32001` | Type error | 类型错误（详见 result.diagnostics） |
| `-32002` | Timeout | 编译超时 |

---

### API 相关前瞻扩展

| 扩展 | 关联 | 方案 |
|------|------|------|
| **F05 持续编译** | 新增 `/ws/v1/watch` 端点 + DevServer | [F05](./future-extensions/F05-Continuous-Compilation.md) |
| **F07 Compiler Marketplace** | 新增插件注册表 API + OpenAPI 自动生成 | [F07](./future-extensions/F07-Compiler-Marketplace.md) |
