# 提示词 13 · S-1 产品组合矩阵看板（前端）

> 前置：`00-master-prompt.md` + `02-frontend-setup.md`。  
> 参考：`docs/04-dashboards/01-s1-product-portfolio.md`（完整 UI Spec）。

---

## 提示词正文

```
你是 Vue3 + ECharts 工程师。请基于完整 UI Spec 实现 S-1 产品组合矩阵看板。

【参考文档】

完整 UI Spec 在 docs/04-dashboards/01-s1-product-portfolio.md，
请严格按 Spec 实现所有字段、交互、状态。

【模块清单】

A. 顶部导航条（面包屑 + 收藏 + 帮助 + 更新时间 + 反馈）
B. 筛选器栏 + 操作按钮（时间/对比/产品/口径/状态 + 切换视图/导出/订阅/分享/记录决策）
C. KPI 摘要条（4 张卡：产品总数 / 总营收 / 加权 ROI / 健康产品比例）
D. 矩阵图（核心 - ECharts scatter 自定义象限）
E. ROI 排行表（Ant Design Vue Table）
F. AI 决策建议（3 张 AiSuggestionCard）
G. 历史矩阵对比（Carousel 4 季度缩略）
H. 底部固定栏（AssistantBubble 浮窗 + 记录决策 + 分享）

【代码组织】

```
src/features/dashboard-s1/
├── views/
│   └── DashboardS1.vue                 # 主视图
├── components/
│   ├── HeaderBar.vue
│   ├── FilterBar.vue
│   ├── KpiSummary.vue
│   ├── PortfolioMatrix.vue             # ★ 矩阵图核心
│   ├── RoiRankingTable.vue
│   ├── AiSuggestionList.vue
│   ├── HistoryComparison.vue
│   └── ActionFooter.vue
├── composables/
│   ├── useS1Data.ts                    # 数据获取
│   ├── useS1Filters.ts                 # 筛选器状态
│   └── useS1Actions.ts                 # 操作行为
├── stores/
│   └── s1.ts                           # Pinia store
├── services/
│   └── s1.api.ts                       # API 调用
├── types/
│   └── s1.types.ts                     # TypeScript 类型
└── index.ts
```

【关键实现要点】

### PortfolioMatrix.vue（核心）

```vue
<template>
  <div class="portfolio-matrix">
    <BaseChart
      type="scatter"
      :option="chartOption"
      :loading="loading"
      :height="600"
      @click="handleBubbleClick"
      @contextmenu="handleBubbleContextMenu"
    />
  </div>
</template>

<script setup lang="ts">
// 关键逻辑：
// 1. 象限背景自定义渲染（graphic 元素绘制四象限背景）
// 2. 气泡大小：sizeRange + 对数刻度
// 3. 气泡颜色：按 healthScore 红/黄/绿
// 4. Tooltip：自定义内容（产品名/营收/增速/ROI/健康分/责任PM/象限）
// 5. 轨迹绘制：hover 时显示过去 4 季度位置（lineSeries）
// 6. 拖拽边界：通过 graphic 实现，调整阈值后 emit
// 7. 数据稀疏处理：< 3 产品改用列表
</script>
```

### 数据类型定义

```ts
// types/s1.types.ts
export interface PortfolioProduct {
  productId: string;
  name: string;
  ownerPm?: User;
  revenue: number;
  manMonths: number;
  roi: number;
  growth: number;
  relativeAdvantage: number;  // x
  healthScore: number;        // color
  retention: number;
  quadrant: 'star' | 'question' | 'cash_cow' | 'dog';
  trajectory?: Array<{ quarter: string; x: number; y: number }>;
  strategicMark?: 'focus' | 'watch' | 'default';
}

export interface S1Data {
  kpi: {
    totalProducts: number;
    totalRevenue: number;
    weightedRoi: number;
    healthRatio: number;
    deltas: Record<string, number>;
  };
  matrix: PortfolioProduct[];
  ranking: PortfolioProduct[];
  suggestions: AiSuggestion[];
  history: QuarterSnapshot[];
}
```

### 筛选器联动

- Pinia store 存筛选状态
- 任一筛选变化 → 重新拉取数据
- URL 参数化（可分享）
- 个人偏好保存

### 状态处理（必须实现 7 种）

| 状态 | 处理 |
|---|---|
| 首次进入无数据 | Empty + 引导按钮 |
| 当前筛选无数据 | Empty + [清空筛选] |
| 数据过期 | 顶部 Alert banner |
| 加载中 | 骨架屏，3s 后提示 |
| 加载失败 | Result + [重试] |
| 无权限 | Result 403 + [申请权限] |
| 部分数据缺失 | 行/气泡 "—" + hover 提示 |

### 响应式

- xxl (≥ 1920) 全屏标准
- xl (1280-1920) 矩阵 65% / 排行 35%
- lg (1024-1280) 矩阵 70% / 排行下方
- < 1024 提示 "请用 PC 端访问"

【交互行为（必须实现）】

矩阵图：
- Hover 气泡：Tooltip 详情
- Click 气泡：跳产品业务模型看板（新标签）
- 右键气泡：菜单（查看明细 / 加入观察 / 发起评审 / 圈定决策）
- 双击气泡：进产品 360°
- Hover 象限：高亮 + 排行表自动筛选
- Hover 气泡：显示 4 季度轨迹拖尾
- 拖拽象限线：临时调阈值，可保存偏好

排行表：
- Hover 行：行变色 + 矩阵气泡高亮跳动
- Click 行：矩阵聚焦该气泡
- 多选（Shift）：浮出"对比"按钮
- 操作列 [详情][评审]

AI 建议卡：
- [发起评审]：创建评审会议
- [详细分析]：唤起助理浮窗
- [记录决策]：唤起决策追溯库表单
- [不采纳]：反馈框

【数据 Mock】

开发时使用 MSW 或 vite-plugin-mock：

```ts
// mock/s1.ts
export const s1Mock = {
  kpi: { totalProducts: 12, totalRevenue: 420000000, ... },
  matrix: [
    { productId: 'p1', name: 'SaaS-发票', revenue: 120000000, growth: 0.45, ... },
    ...
  ],
  ...
};
```

【单元测试】

```
tests/unit/features/dashboard-s1/
├── PortfolioMatrix.spec.ts
├── RoiRankingTable.spec.ts
├── useS1Data.spec.ts
└── ...
```

测试场景：
- 数据渲染正确
- 象限分类正确
- 交互（click/hover/right-click）正常触发
- 筛选联动
- 7 种状态正常显示
- 权限控制

【验收】

1. UI 与 Spec 100% 一致
2. 所有交互可正常触发
3. 7 种状态正常处理
4. 响应式 4 个断点正常
5. 单测覆盖 ≥ 80%
6. Lighthouse 性能 ≥ 90
7. 首屏 < 2s

【交付物】

- 完整 Vue3 + TS 代码
- Pinia store
- Mock 数据
- 单元测试
- README.md（含开发/部署）

【禁止】

- 禁止 any
- 禁止 console.log（debug 除外）
- 禁止内联样式（动态除外）
- 禁止组件内 axios
- 禁止跳过权限校验

现在请生成完整代码。
```
