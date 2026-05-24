# SaaS LCM 模型（Lifecycle Management）

> 业务本质：**订阅经济**。健康定义：NRR ≥ 110% + Adoption ≥ 60% + CSAT ≥ 4.5（**不是** DAU/MAU）。

## 一、客户全生命周期因果链

```
                  CAC
                   │
     ┌─────────────▼─────────────────────────────────────────┐
Lead─►MQL─►SQL─►Trial─►Paid──►Activated──►Adopted──►Expanded─┐
                          │       │           │              │
                          ▼       ▼           ▼              ▼
                       Onboarded TTV短     Cross-sell    Renewed/Advocate
                                              │              │
                              ↓ 任一节点失败           ↓
                          Churned ◄──────────── Downgraded
                              │
                              ▼
                          Win-back

           关键比率：CAC、LTV、LTV/CAC、Payback Period、NRR、GRR
```

## 二、9 个阶段与关键节点

| # | 阶段 | 定义 | 关键指标 | 异常信号 | 干预动作 |
|---|---|---|---|---|---|
| 1 | Lead | 进入获客漏斗 | 来源、质量分 | 渠道转化下降 | 重审渠道预算 |
| 2 | MQL | 营销合格线索 | MQL 转化率 | BANT 通过率下降 | 销售培训 |
| 3 | SQL | 销售合格线索 | SQL 转化率 | 销售跟进周期长 | 销售流程优化 |
| 4 | Trial | 进入试用 | Trial 启动数 | 试用启动放弃 | Onboarding 改造 |
| 5 | Paid | 完成首付 | Trial→Paid 转化率 | 转化率下降 | Trial 智能引导 |
| 6 | Activated | 完成关键模块开通 | 激活率 | 部署门槛高 | CS 介入 + 简化部署 |
| 7 | Adopted | 深度使用核心模块 | Adoption Score、Seat 激活率 | 关键模块未用 | Playbook 推进 |
| 8 | Expanded | 加购/升级 | 扩张率、NRR | 用量接近上限不升级 | 销售触达 |
| 9 | Renewed/Advocate | 续约 + 推荐 | 续约率、NPS | 健康分骤降 | CSM 上门 |

## 三、5 大核心 KPI（业务模型层级）

| KPI | 公式 | 健康阈值 | 衡量什么 |
|---|---|---|---|
| **CAC** | (市场+销售投入) / 新增付费客户 | 视模式 | 获客效率 |
| **LTV** | 月毛利 / 月流失率 | 越高越好 | 客户终身价值 |
| **LTV/CAC** | LTV / CAC | ≥ 3 | 商业可持续 |
| **NRR** | (期初 ARR + 扩张 - 收缩 - 流失) / 期初 ARR | ≥ 110% | 现有客户增长 |
| **GRR** | (期初 ARR - 收缩 - 流失) / 期初 ARR | ≥ 90% | 留存底线 |

## 四、客户健康分模型

```
客户健康分 = 0.30 × 采纳深度
           + 0.20 × 活跃度
           + 0.15 × Admin 健康
           + 0.15 × 续约信号（合同前指数 + 商务关系）
           + 0.10 × CSAT
           + 0.10 × 商业关系（NPS + 投诉记录）

分级：
  ≥ 80：绿，健康
  60-79：黄，关注
  < 60：红，风险
```

## 五、典型流失原因分类

| 类别 | 占比标杆 | 应对 |
|---|---|---|
| 产品不匹配 | 25% | 产品改进 / 转介绍 |
| 价格 | 20% | 定价策略 / 折扣 |
| 竞品 | 15% | 差异化 / 高管沟通 |
| 客户业务变化 | 15% | 暂停而非流失 |
| 服务问题 | 10% | CS 流程改进 |
| 财务问题 | 10% | 分期 / 弹性付款 |
| 其他 | 5% | — |

## 六、关键干预点

| 阶段 | 干预 Playbook | 触发条件 |
|---|---|---|
| Trial → Paid | P-SAC-001 Trial 智能引导 | 转化率周环比 ↓ ≥ 3pp |
| 首日激活 | P-SAC-003 首日激活闯关 | 注册后即启动 |
| Onboarding | P-SAC-002 Onboarding 加速 | TTV ↑ ≥ 30% |
| 采纳 | P-SAD-001 核心模块采纳推进 | 核心模块未用 |
| 扩张 | P-SE-003 扩张机会激活 | 用量 ≥ 80% |
| 续约 | P-SE-002 续约 90 天 Journey | 到期前 90 天 |
| 流失风险 | P-SH-001 流失挽留 | 健康分 < 50 |

## 七、看板对应

详见 [B-1 SaaS LCM 看板](../04-dashboards/06-b1-saas-dashboard.md)。

## 八、指标清单速查（22 个）

| 类别 | 指标 |
|---|---|
| 获客 (3) | M-SA-001 CAC / M-SA-002 LTV/CAC / M-SA-003 Payback |
| 激活 (4) | M-SAC-001 Trial→Paid / M-SAC-002 TTV / M-SAC-003 首日激活 / M-SAC-004 Onboarding 完成率 |
| 采纳 (4) | M-SAD-001 核心模块采纳率 / M-SAD-002 Seat 激活率 / M-SAD-003 Adoption Score / M-SAD-004 粘性 |
| 扩张续约 (5) | M-SE-001 NRR / M-SE-002 GRR / M-SE-003 续约率 / M-SE-004 扩张率 / M-SE-005 续约提前指数 |
| 留存 (3) | M-SR-001 Logo 留存 / M-SR-002 流失原因 / M-SR-003 预警准确率 |
| 客户健康 (3) | M-SH-001 健康分 / M-SH-002 NPS / M-SH-003 CSAT |

详细 SQL 见 [指标 SQL/DSL](./05-metrics-sql.md)。
