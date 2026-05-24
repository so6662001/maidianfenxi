# 提示词 14 · S-2 至 S-5 战略看板

> 前置：`00-master-prompt.md` + `02-frontend-setup.md` + `13-dashboard-s1.md`。  
> 参考：`docs/04-dashboards/02-s2-strategic-health.md` 至 `05-s5-decision-pool.md`。

---

## 提示词正文

```
你是 Vue3 + ECharts 工程师。请基于 UI Spec 实现 S-2 至 S-5 四个战略看板。

【整体要求】

1. 复用 S-1 已建立的基础设施（设计 Tokens、共享组件、Mock 工具等）
2. 每个看板独立 feature 目录
3. 共享组件提取到 src/shared/components/
4. 遵循 S-1 同样的代码组织、状态、响应式、测试规范

【4 张看板模块清单】

### S-2 战略健康度雷达
src/features/dashboard-s2/
- HeaderBar + FilterBar
- KpiHorizontalBar（10 个数字横向滚动）
- HealthRadar（ECharts radar 三产品对比）
- ProductHealthBars（条形）
- HealthTrendLine（4 周折线）
- AnomalyTopTable（异动 TOP10，行可展开 AnomalyCard）
- WeeklyFocusCards（DecisionCard ×3）

### S-3 PM 效能榜
src/features/dashboard-s3/
- HeaderBar + FilterBar（产品线/角色/周期/红黄灯）
- CompanyKpis（6 个总览）
- PmRankingTable（虚拟滚动 100+ 行）
- TeamComparisonBar
- SkillRadarDrawer（点击展开 mini radar）
- AiOrgInsights（Alert + Timeline）

### S-4 战略假设追踪
src/features/dashboard-s4/
- HeaderBar
- TopProgressStats（年度进度条 + 状态统计）
- HypothesisWall（瀑布流卡片墙）
  - HypothesisCard（核心组件）
- HypothesisFormDrawer（新增/编辑表单）

### S-5 决策待办池
src/features/dashboard-s5/
- GreetingHeader（问候 + 统计）
- UrgentDecisionList（UrgentDecisionCard ×3 大卡）
- ImportantList（行列表）
- ThisWeekDecidedList（已决策列表）
- AssistantInputBar（固定底部）

【共享组件（提取到 src/shared/components/）】

- `<KpiCard>`（小/中/大三尺寸）
- `<AiSuggestionCard>`
- `<AnomalyCard>`
- `<DecisionCard>`
- `<UrgentDecisionCard>`（S-5 专用）
- `<HypothesisCard>`（S-4 专用）
- `<AssistantBubble>`（全局浮窗）
- `<BaseChart>`（封装 ECharts）

每个共享组件必须：
- 强类型 Props（TS interface）
- 完整 emits 定义
- 提供 sm/md/lg 多尺寸
- 支持 loading / error / empty 状态
- 单测覆盖

【数据 API（统一规约）】

```
GET /api/dashboard/s2/health        Query: timeRange, compareBase
GET /api/dashboard/s3/pm-performance Query: period, productLine, role
GET /api/dashboard/s4/hypotheses     Query: year, status
GET /api/dashboard/s5/decisions      Query: userId
POST /api/dashboard/s5/decide        Body: { itemId, choice, comment }
POST /api/dashboard/s5/transfer      Body: { itemId, transferTo, comment }
```

返回详见 docs/04-dashboards/。

【关键交互】

### S-2 异动表格
- 行 click → 展开 AnomalyCard（含归因详情 + 操作按钮）
- [处理][圈人][提单] 触发对应 Service 调用 + 二次确认

### S-3 PM 行
- Click → 抽屉显示个人详情（雷达 + 历史决策列表）
- 权限校验（PM 本人 vs 上级 vs 高管 不同权限）

### S-4 假设卡片
- [查看明细] → 抽屉显示完整数据
- [发起调整评审] → 创建评审会议
- [标记关键风险] → 高亮 + 通知

### S-5 紧急决策
- [选 A/B/C] → 二次确认 → 调用 decide API → 自动登记决策追溯
- 决策后卡片消失，统计更新
- 完成所有 3 件后显示"今日决策完成 🎉"

【AssistantBubble 全局集成】

在 DefaultLayout 中挂载，所有看板共享：

```vue
<template>
  <a-layout>
    <a-layout-sider>...</a-layout-sider>
    <a-layout>
      <a-layout-header>...</a-layout-header>
      <a-layout-content>
        <router-view />
      </a-layout-content>
    </a-layout>
    <!-- 全局浮窗 -->
    <AssistantBubble :context="currentRouteContext" />
  </a-layout>
</template>
```

【权限矩阵】

各看板严格按 docs/01-requirements/02-user-roles.md 实现：
- S-1：总 PM / CEO / CFO 全部；产品线负责人本线
- S-2：基本同上
- S-3：仅总 PM / CEO 看全部；产品线负责人本线；PM 本人详情
- S-4：高管 + 关联人
- S-5：仅本人

【状态规范】

每个看板都必须实现 7 种状态：
- 首次无数据
- 筛选无数据
- 数据过期
- 加载中
- 加载失败
- 无权限
- 部分数据缺失

【单元测试】

每个看板：
- 主视图渲染
- 交互（click/hover）
- 状态切换
- 权限控制
- Mock 数据下完整跑通

覆盖率 ≥ 80%。

【验收】

1. 4 个看板 UI 与各自 Spec 100% 一致
2. 共享组件可独立使用
3. 全局 AssistantBubble 在每个看板可用
4. 7 种状态全部正常
5. 单测覆盖 ≥ 80%
6. 首屏 < 2s

【交付物】

- 4 个看板完整代码
- 共享组件（含 Storybook 文档）
- Pinia stores
- Mock 数据
- 单元测试
- README.md

【禁止】

同 13-dashboard-s1.md。

现在请生成。
```
