# 提示词 15 · B-1 至 B-3 业务模型看板

> 前置：`00-master-prompt.md` + `02-frontend-setup.md`。  
> 参考：`docs/04-dashboards/06-b1` ~ `08-b3` + `09-g1`。

---

## 提示词正文

```
你是 Vue3 + ECharts 工程师。请实现 B-1（SaaS LCM）、B-2（AI 单位经济）、B-3（平台双边流动性）、G-1（用户增长决策中枢）4 个看板。

【整体要求】

1. 复用 S-1/S-2/S-5 已建的共享组件、设计 Tokens、Mock 工具
2. 每个看板独立 feature 目录
3. 业务模型的复杂可视化（漏斗 / 桑基 / 热力图 / 矩阵）必须用 ECharts 实现

【B-1 SaaS LCM 看板】

src/features/dashboard-b1/
- HeaderBar + FilterBar（时间/对比/产品线/规模/行业/销售年份/套餐/CSM）
- LcmFunnel（9 阶段漏斗：Lead→MQL→SQL→Trial→Paid→Activated→Adopted→Expanded→Renewed）
- NrrCard（大号 KpiCard + 趋势小图 + hover 拆解）
- HealthSankey（节点=健康分段，流向=阶段间迁移）
- RenewalRiskMap（自研网格，X=到期周/Y=健康分/气泡=ARR）
- ModuleAdoptionHeatmap（行=模块, 列=客户规模）
- SeatActivationScatter
- ChurnAnalysis（词云 + 桑基）
- AiSuggestionList（3 张）

关键交互：
- 健康桑基点击红色 → 自动启动 P-SH-001
- 续约风险地图气泡 → 客户 360°
- 续约风险地图多选 → 一键派单 CSM

【B-2 AI 单位经济看板】

src/features/dashboard-b2/
- HeaderBar + FilterBar（时间/模型版本/场景/用户类型/套餐/渠道）
- EconomicsThreeCards（单次毛利 / 价值采纳率 / TTFV 大号卡）
- MarginTrendChart（折线 + 堆叠 area 成本拆解）
- CapabilityGapMatrix（核心：意图 × 满意度 热力图）
- QualityThreeCharts（点踩 / 拒答 / 重写 趋势 + 异动标记）
- RetentionAndConversion（留存矩阵 + 漏斗）
- AiSuggestionList

关键交互：
- 能力缺口矩阵 → 点击格 → 展开示例 Prompt + 启动 P-AQ-002
- 单次毛利异常 → 一键启动 P-AU-001
- 质量图表点击异常点 → 弹窗显示近期 Prompt 列表

【B-3 平台双边流动性看板】

src/features/dashboard-b3/
- HeaderBar + FilterBar（时间/类目层级/地区/商家等级/买家等级）
- KeyMetricsFourCards（24h 响应率 / 撮合成功率 / 双边活跃比 / GMV）
- SupplyDemandHeatmap（核心：类目 × 地区 × 供需比）
- ImbalanceTopTable（失衡 TOP10）
- BuyerFunnel / MerchantFunnel
- NetworkEffectChart（双轴折线：商家增量 vs 买家活跃）
- Commercialization（GMV 趋势 + 会员分布 + 流量包 ROI 散点）
- AiSuggestionList

关键交互：
- 供需热力图红色格 → 弹出招商/导流决策面板
- 失衡 TOP10 → 一键启动 P-B-001
- 询盘流动性预警 → 一键 P-L-001

【G-1 用户增长决策中枢】

src/features/dashboard-g1/
- HeaderBar + FilterBar（产品/周期）
- GrowthLoopsHealthBar（增长循环健康度，按产品）
- SegmentMigrationSankey
- BottlenecksIdentification（TOP 3 瓶颈 + 推荐动作）
- GrowthOpportunityCards
- RunningExperimentsList
- PlaybookLibrary（32 个 Playbook 卡片）
- UserPanoramicEntry（按企业/用户/分群搜索入口）

关键交互：
- 瓶颈卡 [发起评审 / 详细分析]
- 机会卡 [一键发起实验] → 跳实验平台
- Playbook 卡 → 启动确认 + 实验设计向导

【共享组件升级】

新增（提取到 src/shared/components/）：
- `<FunnelChart>` 漏斗封装
- `<SankeyChart>` 桑基封装
- `<HeatmapChart>` 热力图封装
- `<MetricGrid>` 网格组件（用于风险地图）
- `<OpportunityCard>` 增长机会卡

【数据 API】

```
GET /api/dashboard/b1/saas
GET /api/dashboard/b2/ai
GET /api/dashboard/b3/platform
GET /api/dashboard/g1/growth
```

各返回结构详见各看板 UI Spec。

【关键技术点】

### 大数据量渲染优化

- 桑基图节点 > 50 时聚合
- 热力图 cell > 1000 时虚拟化
- 表格 100+ 行虚拟滚动
- 图表数据 transformer worker 处理（避免主线程阻塞）

### 状态规范 + 响应式

同 S-1，7 种状态 + 4 个断点。

### 权限

每个看板按角色权限严格控制：
- B-1：SaaS 产品线 PM / CSM / 销售
- B-2：AI PM / Prompt 工程师 / CFO
- B-3：平台运营 / 招商 / 风控
- G-1：增长团队 / 运营

跨产品高管可看全部。

### 性能

- 首屏 < 2s
- 单图查询 < 5s
- 大数据渲染（10w+ 点）平滑

【单元测试】

每个看板单元测试 + 关键交互 E2E。

【验收】

1. 4 个看板 UI 与 Spec 100% 一致
2. 业务可视化（漏斗/桑基/热力图）正确渲染
3. 关键交互正常触发 Playbook
4. 7 种状态全部正常
5. 性能达标
6. 单测 ≥ 80% 覆盖

【交付物】

- 4 个看板完整代码
- 新增共享组件
- Pinia stores
- Mock 数据（含大数据量场景）
- 单元测试 + E2E
- README.md

【禁止】

- 禁止 3D 饼图、过度装饰
- 禁止超过 5 段的环形
- 禁止彩虹色图表（必须遵循设计 Tokens）
- 同总纲

现在请生成。
```
