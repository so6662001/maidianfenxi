# B-1 SaaS LCM 业务模型看板 · UI Spec

> **服务对象**：SaaS 产品线负责人、CSM 负责人、销售负责人  
> **数据频率**：T+1 + 续约日实时

## 布局

```
Row 1 (头部 + 筛选)                        高 112px
Row 2 (LCM 漏斗 + NRR 卡)                  高 320px
  ├ LCM 9 阶段漏斗            14 列
  └ NRR/GRR 大数 + 趋势        10 列
Row 3 (健康分布 + 续约风险地图)             高 360px
  ├ 健康桑基（阶段迁移）       12 列
  └ 续约风险地图（90 天滚动） 12 列
Row 4 (核心模块采纳热力 + Seat 激活)       高 280px
Row 5 (流失分析 + AI 洞察)                 高 280px
```

## 模块

### LCM 漏斗（9 阶段）

- 组件：ECharts funnel 自定义
- 阶段：Lead → MQL → SQL → Trial → Paid → Activated → Adopted → Expanded → Renewed
- 点击节点 → 该节点用户/企业列表 + Playbook 启动入口

### NRR / GRR 卡

- `<KpiCard>` 大号
- hover 显示拆解：扩张 / 收缩 / 流失
- 趋势小图 12 周

### 健康分桑基

- 节点：红 / 黄 / 绿 健康分段
- 流向：每周间的迁移
- 点击红色区域 → 自动跳 P-SH-001 流失挽留 Playbook 启动

### 续约风险地图

- 自研网格组件
- X：到期周（未来 13 周）
- Y：客户健康分（5 段）
- 气泡大小：ARR
- 颜色：风险等级
- 点击气泡 → 客户 360° 画像
- 多选 → 一键派单 CSM

### 核心模块采纳热力

- ECharts heatmap
- 行：模块（contract / invoice / crm / ...）
- 列：客户规模（小/中/大）
- 颜色：采纳率
- 点击格 → 该客户群列表

### Seat 激活散点

- X：购买 Seat 数
- Y：激活率
- 气泡：ARR
- 颜色：健康分

### 流失分析

- 词云：流失原因关键词
- 桑基：流失前 30 天行为路径

### AI 洞察

- `<AiSuggestionCard>` ×3
- 自动生成"本季 SaaS 关键洞察"

## 筛选器

```
时间周期 | 对比基准 | 产品线 (SaaS-发票/合同/CRM) | 客户规模 | 行业 | 销售年份 | 套餐 | 责任 CSM
```

## 关键交互闭环

- 红色客户 → 一键启动 P-SH-001
- 即将到期客户 → 一键派单 CSM
- 任何客户气泡 → 进入企业 360°

## 数据接口

```
GET /api/dashboard/b1/saas
  Query: filters
  Response: {
    funnel: [...9 stages with counts],
    nrr: { current, trend, breakdown },
    grr: { ... },
    healthSankey: [...],
    renewalRiskMap: [{ orgId, expireWeek, healthScore, arr, risk }],
    moduleAdoption: [[...heatmap]],
    seatActivation: [...],
    churnAnalysis: { reasons, paths },
    aiInsights: [...3]
  }
```
