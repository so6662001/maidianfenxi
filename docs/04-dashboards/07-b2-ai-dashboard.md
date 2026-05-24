# B-2 AI 单位经济业务模型看板 · UI Spec

> **服务对象**：AI 产品 PM、Prompt 工程师、CFO  
> **数据频率**：实时（成本敏感）

## 布局

```
Row 1 (头部 + 筛选)                              高 112px
Row 2 (经济三件套大数)                          高 200px
  ├ 单次毛利     ├ 价值采纳率     ├ TTFV
Row 3 (单位经济演进 + 能力缺口矩阵)             高 400px
  ├ 单次毛利 + 成本拆解趋势        12 列
  └ 能力缺口热力（意图 × 满意度）  12 列
Row 4 (质量三件套)                              高 280px
  ├ 点踩率趋势  ├ 拒答率趋势  ├ 重写率趋势
Row 5 (留存与转化 + AI 洞察)                    高 280px
```

## 模块

### 经济三件套大数

- `<KpiCard>` ×3 大号
- 单次毛利：hover 显示按场景/模型/套餐拆解
- 价值采纳率：hover 显示按行为拆解（复制/导出/追问）
- TTFV：hover 显示分布（< 1 min / 1-3 / 3-10 / 10+）

### 单次毛利演进 + 成本拆解

- 组件：ECharts line + 堆叠 area
- 折线：单次毛利
- 堆叠：Token in / Token out / 基础设施 / 其他
- 标注：模型版本上线点
- 缩放：支持时间段聚焦

### 能力缺口矩阵（核心）

- 组件：ECharts heatmap
- X：意图聚类（LLM Embedding + 聚类）
- Y：场景/角色
- 颜色：采纳率（低）/ 点踩率（高）
- 点击格 → 展开该意图的示例 Prompt + 案例
- 一键 → 启动 P-AQ-002 能力缺口攻坚

### 点踩率 / 拒答率 / 重写率

- ECharts line + 异动标记
- 点击异常点 → 弹窗显示近期相关 Prompt 列表

### 留存与转化

- 留存矩阵（7 日 / 30 日）
- 免费→付费漏斗
- 用户分群对比

### AI 洞察

- 模型路由建议（P-AU-001 触发）
- 能力缺口攻坚建议（P-AQ-002）

## 筛选器

```
时间周期 | 模型版本 | 场景 | 用户类型 | 套餐 | 渠道
```

## 数据接口

```
GET /api/dashboard/b2/ai
  Query: filters
  Response: {
    economics: { sessionMargin, valueAdoption, ttfv },
    marginTrend: [...with cost breakdown],
    capabilityGap: [[...heatmap]],
    quality: { thumbsdown, refusal, rewrite },
    retention: { matrix, conversion },
    aiInsights: [...]
  }
```
