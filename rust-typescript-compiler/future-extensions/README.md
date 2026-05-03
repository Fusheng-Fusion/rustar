# Future Extensions — BlockType 前瞻性扩展方案

> **文档版本**：v1.0 | **日期**：2026-05-02 | **状态**：提议草案
> **定位**：在现有 BlockType v3.0 架构之上，追加 12 个前瞻性功能方案

---

## 索引

| # | 功能 | 优先级 | 复杂度 | 建议 Phase | 依赖 |
|---|------|--------|--------|-----------|------|
| [F01](./F01-Distributed-Build-Cache.md) | 分布式编译缓存 | 🔴 高 | 高 | Phase 2-3 | bt-query, bt-cargo |
| [F02](./F02-Supply-Chain-Security.md) | 软件供应链安全 | 🔴 高 | 中 | Phase 2-3 | bt-cargo |
| [F03](./F03-Formal-Verification.md) | 形式化验证集成 | 🟡 中 | 高 | Phase 5+ | bt-ir, bt-dialect-rust |
| [F04](./F04-Compile-Time-Fuzzing.md) | 编译时 Fuzzing | 🟡 中 | 中 | Phase 5+ | bt-ir, bt-passes |
| [F05](./F05-Continuous-Compilation.md) | 持续编译 | 🔴 高 | 中 | Phase 4 | bt-api, bt-query, EventStore |
| [F06](./F06-AI-Adaptive-Strategy.md) | AI 自适应编译策略 | 🟡 中 | 中 | Phase 4-5 | bt-ai, EventStore |
| [F07](./F07-Compiler-Marketplace.md) | Compiler Marketplace | 🟡 中 | 中 | Phase 5+ | bt-plugin-host, bt-api |
| [F08](./F08-Green-Compilation.md) | 绿色编译 | 🟢 低 | 低 | Phase 4 扩展 | bt-telemetry |
| [F09](./F09-Cross-Compilation-Management.md) | 交叉编译环境管理 | 🟡 中 | 低 | Phase 3 | bt-std-bridge |
| [F10](./F10-Browser-Compiler.md) | 浏览器内编译器 | 🟢 低 | 高 | Phase 5+ | bt-core, bt-ir (无 rustc 依赖部分) |
| [F11](./F11-Compiler-LLM-Collaboration.md) | 编译器 LLM 协作 | 🟢 低 | 中 | Phase 5+ | bt-ai, EventStore |
| [F12](./F12-Cross-Language-Reflection.md) | 跨语言编译时反射 | 🟢 低 | 高 | Phase 5+ | bt-ir, bt-dialect-core |

---

## 推荐整合策略

### 立即纳入（当前路线图无冲突）
- **F05 持续编译** → 接入 Phase 4 的 LSP + WebSocket 基础设施
- **F09 交叉编译管理** → 扩展 Phase 3 的 bt-std-bridge
- **F08 绿色编译** → Phase 4 的 bt-telemetry 扩展

### Phase 2-3 可整合
- **F01 分布式缓存** → bt-query Salsa 引擎天然支持远程后端
- **F02 供应链安全** → bt-cargo 依赖解析阶段即可插入

### Phase 5+ 储备
- 其余 7 个功能保持为远期方向，不影响当前 45 周交付计划

---

## 与现有架构的关系

所有 12 个功能遵循相同的设计约束：
1. **不修改现有 crate 的核心职责** — 通过新增 crate / 扩展 trait 实现
2. **向后兼容** — 现有 API 签名不变
3. **可选启用** — 通过 bt-cli 参数 / 配置开关控制
4. **渐进式交付** — 每个功能分 1-3 个 MVP 阶段，确保每阶段可测试
