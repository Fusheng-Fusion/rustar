# F08 — 绿色编译（Green Compilation）

> **优先级**：🟢 低 | **复杂度**：低 | **建议 Phase**：Phase 4 扩展 | **工期**：~2 周

---

## 1. 概述

编译器每次运行都消耗 CPU 资源和电力。在碳中和和 ESG 合规的大趋势下，编译过程的碳足迹管理变得有意义。本方案为 BlockType 增加**碳排放追踪 + 绿色调度 + 增量编译激励**功能，让开发者了解每次编译的环境成本。

### 核心理念

```
- 编译器应该知道自己的能源消耗
- 增量编译不仅是"快"，更是"环保"
- 绿色调度可以根据电网碳强度自动决定何时编译
```

---

## 2. 架构设计

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│               bt-telemetry 扩展（绿色编译模块）                    │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CarbonTracker                             │     │
│  │  - 编译碳足迹估算（CPU 时间 × 功耗模型 × 碳强度）           │     │
│  │  - 功耗模型：基于 CPU TDP + 利用率                         │     │
│  │  - 碳强度：本地数据库 / 外部 API 查询                       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    GreenScheduler                            │     │
│  │  - 将非紧急编译推迟到低碳时段                              │     │
│  │  - 夜间 / 周末 / 可再生能源充足时段                         │     │
│  │  - 通过 ElectricityMap API 查询实时碳强度                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                    CompileReport                             │     │
│  │  - 编译结束后显示碳足迹报告                                │     │
│  │  - 全量编译 vs 增量编译的碳节省                            │     │
│  │  - 可导出为 ESG 报告格式                                   │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘

碳排放估算模型：
  编译碳排(gCO₂) = CPU_时间(秒) × 平均功率(W) × 碳强度(gCO₂/kWh) / 1000
```

### 2.2 核心类型定义

```rust
// crates/bt-telemetry/src/carbon.rs

/// 碳足迹追踪器
pub struct CarbonTracker {
    /// CPU 平均功耗（W），从系统获取或默认值
    cpu_power_watts: f64,
    /// 区域碳强度（gCO₂/kWh）
    carbon_intensity: f64,
    /// 是否启用外部 API
    use_api: bool,
}

impl CarbonTracker {
    /// 估算编译任务的碳足迹
    pub async fn estimate(&self, duration_ms: u64, cpu_utilization: f64) -> CarbonEstimate {
        let hours = duration_ms as f64 / 3_600_000.0;
        let energy_kwh = (self.cpu_power_watts * cpu_utilization) * hours / 1000.0;
        let emissions_g = energy_kwh * self.carbon_intensity;

        CarbonEstimate {
            duration_ms,
            energy_kwh,
            carbon_intensity: self.carbon_intensity,
            emissions_g: emissions_g.round(),
            equivalent: CarbonEquivalent {
                km_driven: emissions_g / 120.0,      // 相当于开车 X 公里
                phone_charges: emissions_g / 0.5,     // 相当于充 X 次手机
                tree_days: emissions_g / 20.0,         // 相当于 X 棵树的日吸收量
            },
        }
    }

    /// 从 ElectricityMap API 获取实时碳强度
    pub async fn fetch_carbon_intensity(&mut self, zone: &str) -> Result<f64, CarbonError> {
        let url = format!("https://api.electricitymap.org/v3/carbon-intensity/latest?zone={zone}");
        let response = reqwest::get(&url).await?;
        let data: CarbonIntensityResponse = response.json().await?;
        Ok(data.carbon_intensity)
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct CarbonEstimate {
    pub duration_ms: u64,
    pub energy_kwh: f64,
    pub carbon_intensity: f64,
    pub emissions_g: f64,
    pub equivalent: CarbonEquivalent,
}

#[derive(Debug, Clone, Serialize)]
pub struct CarbonEquivalent {
    pub km_driven: f64,        // 相当于驾驶汽车的公里数
    pub phone_charges: f64,    // 相当于充手机次数
    pub tree_days: f64,        // 相当于棵树一天的吸收量
}

/// 绿色调度器
pub struct GreenScheduler {
    tracker: CarbonTracker,
    /// 碳强度阈值（超过此值推迟编译）
    threshold: f64,
}

impl GreenScheduler {
    /// 检查当前是否适合编译
    pub async fn should_compile(&self) -> GreenDecision {
        let intensity = self.tracker.carbon_intensity;
        if intensity > self.threshold {
            GreenDecision::Defer {
                reason: format!("当前碳强度 {:.0} gCO₂/kWh，超过阈值 {:.0}", intensity, self.threshold),
                suggested_at: self.estimate_low_carbon_time().await,
            }
        } else {
            GreenDecision::Compile
        }
    }

    /// 估算低碳时段
    async fn estimate_low_carbon_time(&self) -> Option<DateTime<Utc>> {
        // 根据历史数据或 API 预测
        // 通常是夜间（太阳能不发电时风能多）或周末
        todo!()
    }
}

pub enum GreenDecision {
    Compile,
    Defer { reason: String, suggested_at: Option<DateTime<Utc>> },
}
```

### 2.3 编译碳报告

```
$ bt build --carbon

BlockType Compile Report:
  Duration: 12.3s
  CPU: 4 cores @ 65W avg
  Carbon Intensity: 380 gCO₂/kWh (当前区域电网)

  ─────────────────────────────────────
  🌱 碳足迹
  排放: 0.27 gCO₂ (全量编译)
  ↳ 相当于: 开车 2.3m / 充手机 0.5 次

  如果使用增量编译（估算-30%）:
  可节省: 0.08 gCO₂

  ─────────────────────────────────────
  📊 本次会话累计: 1.2 gCO₂
  本周累计: 4.7 gCO₂ (= 39m 开车)
```

---

## 3. 功能特性

| # | 特性 | 说明 |
|---|------|------|
| C1 | **碳足迹追踪** | 每次编译估算 CO₂ 排放量 |
| C2 | **直观类比** | 开车公里数 / 充手机次数 / 树木吸收量 |
| C3 | **实时碳强度** | 通过 ElectricityMap API 获取电网实时碳强度 |
| C4 | **绿色调度** | 碳强度超标时自动推迟非紧急编译 |
| C5 | **增量编译激励** | 显示增量编译节省的碳排放 |
| C6 | **ESG 报告** | 编译碳足迹可导出为 ESG 报告格式 |
| C7 | **会话统计** | 本周/本月编译累计碳足迹 |

---

## 4. 与现有架构的集成

### 新增模块

| 模块 | 位置 | 行数估算 |
|------|------|---------|
| 碳追踪 | `bt-telemetry/src/carbon.rs` | ~800 |

### 修改的现有 crate

| Crate | 修改内容 | 影响 |
|-------|---------|------|
| `bt-telemetry` | 新增 `carbon.rs` 模块 | 低 |
| `bt-cli` | 新增 `--carbon` 参数 + `bt carbon` 子命令 | 低 |
| `bt-service` | CompileService 编译结束后调用 CarbonTracker | 低（可选） |

### CLI 变更

```bash
# ─── 碳足迹模式 ───

bt build --carbon                    # 编译 + 显示碳足迹
bt build --carbon --green            # 绿色调度模式（高碳时推迟）
bt carbon                            # 显示碳足迹统计
bt carbon --export=esg.json           # 导出 ESG 报告
bt carbon --session                  # 当前会话统计
bt carbon --weekly                   # 本周统计

# ─── 绿色调度 ───

bt build --green                     # 绿色编译（高碳时弹窗确认）
bt build --green --wait               # 绿色编译（高碳时等待到低碳时段）
```

---

## 5. 分阶段实施

### Phase 4 扩展（2 周）

| 周 | 任务 | MVP |
|---|------|-----|
| 1 | CarbonTracker + CPU 功耗估算 + 排放计算 | `bt build --carbon` 显示碳足迹 |
| 2 | GreenScheduler + 碳强度 API 集成 | 绿色调度可用 |

---

## 6. 风险评估

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| 碳强度 API 需要网络 | 中 | 低 | 默认使用本地估算，API 仅作增强 |
| 功耗模型不够精确 | 高 | 低 | 精度 ~30%，趋势有意义即可 |
| 绿色调度影响开发效率 | 低 | 低 | 默认仅报告，不强制调度 |
