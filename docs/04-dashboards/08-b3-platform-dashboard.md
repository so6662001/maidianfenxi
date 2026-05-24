# B-3 平台双边流动性业务模型看板 · UI Spec

> **服务对象**：平台运营负责人、招商团队、风控团队  
> **数据频率**：实时（流动性敏感）

## 布局

```
Row 1 (头部 + 筛选)                              高 112px
Row 2 (流动性命脉 + 双边大数)                   高 240px
  ├ 24h 响应率  ├ 撮合成功率  ├ 双边活跃比  ├ GMV
Row 3 (供需热力地图 ⭐)                          高 480px
  ├ 类目 × 地区热力（左 16 列）
  └ 失衡 TOP10 列表（右 8 列）
Row 4 (双边漏斗 + 网络效应趋势)                 高 320px
  ├ 买家漏斗  ├ 商家漏斗  ├ 网络效应弹性
Row 5 (商业化 + AI 洞察)                        高 280px
```

## 模块

### 4 大数

- `<KpiCard>` ×4
- 24h 响应率：hover 显示按类目拆解 + 趋势
- 撮合成功率：T+30 数据
- 双边活跃比：当日实时
- GMV：日累计

### 供需热力地图（核心）

- 组件：ECharts heatmap（3D 矩阵或自研网格）
- X：类目（三级）
- Y：地区
- 颜色：供需比（红=失衡 / 绿=健康）
- 点击格 → 弹出该类目地区的"招商/导流"决策面板
- 鼠标 hover → 详细数据 + 趋势小图

### 失衡 TOP10 列表

| 列 | 字段 |
|---|---|
| 排名 | — |
| 类目 / 地区 | 文本 |
| 供需比 | 数值 + 健康颜色 |
| 严重度 | P0/P1/P2 |
| 持续天数 | — |
| 影响询盘数 | — |
| 操作 | [启动 P-B-001] |

### 买家漏斗

```
访问 → 搜索 → 详情 → 留线索 → 询价 → 报价 → 成交
```

每步显示转化率 + 标红低于阈值的环节。

### 商家漏斗

```
入驻 → 上架 → 活跃 → 收询盘 → 响应 → 报价 → 成单 → 续费
```

### 网络效应趋势

- ECharts line（双轴）
- 左轴：商家增量
- 右轴：买家活跃增量
- 趋势：弹性系数

### 商业化

- GMV 趋势 + 各类目占比饼
- 会员等级分布 + ARPU
- 流量包 ROI 散点（X：花费 / Y：ROI / 气泡：商家）

### AI 洞察

- 招商建议（基于供需失衡）
- 流动性预警（基于响应率下滑）

## 关键交互

- 供需地图红色格 → 启动招商/导流
- 失衡 TOP10 → 一键 P-B-001
- 询盘流动性预警 → 一键 P-L-001

## 筛选器

```
时间周期 | 类目层级 | 地区 | 商家等级 | 买家等级
```

## 数据接口

```
GET /api/dashboard/b3/platform
  Query: filters
  Response: {
    metrics: { responseRate24h, matchRate, buyerMerchantRatio, gmv },
    supplyDemandMap: [[...heatmap]],
    imbalanceTop10: [...],
    buyerFunnel: [...steps],
    merchantFunnel: [...steps],
    networkEffect: { merchantElasticity, buyerElasticity, trend },
    commercialization: { gmvTrend, membership, trafficRoi },
    aiInsights: [...]
  }
```
