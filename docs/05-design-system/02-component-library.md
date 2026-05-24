# 组件库映射

> **基础库**：Ant Design Vue 4.x + ECharts 5+  
> **自研业务组件**：见下文 5 大核心组件

## 一、基础库映射表

| 业务元素 | Ant Design Vue 组件 | 备注 |
|---|---|---|
| 页面布局 | `Layout` + `Layout.Sider` + `Layout.Content` | 中后台标准 |
| 卡片 | `Card` | hover 加 `--shadow-2` |
| 表格 | `Table` | 启用虚拟滚动 |
| 表单 | `Form` + `Form.Item` | 配 vee-validate 校验 |
| 按钮 | `Button` | type=primary/default/dashed/text/link |
| 时间筛选 | `RangePicker` + 快捷预设 | 含相对时间 |
| 下拉选 | `Select` / `Cascader` | |
| 单选/复选 | `Radio.Group` / `Checkbox.Group` | |
| 标签 | `Tag` | 状态色 |
| Badge | `Badge` | 角标 |
| 头像 | `Avatar` + `Avatar.Group` | |
| 抽屉 | `Drawer` | 详情/配置 |
| 模态 | `Modal` | 二次确认 |
| 提示 | `Tooltip` / `Popover` | |
| Toast | `message` | 操作反馈 |
| 通知 | `notification` | 重要消息 |
| Loading | `Spin` / `Skeleton` | 优先骨架 |
| 空状态 | `Empty` | 含引导 |
| 错误 | `Result` | 含重试 |
| 步骤 | `Steps` | Wizard |
| Tabs | `Tabs` | |
| 树 | `Tree` / `TreeSelect` | 类目选择 |
| Transfer | `Transfer` | 穿梭框 |
| Upload | `Upload` | 文件上传 |
| 引导 | `Tour` | 新功能引导 |
| 反馈 | 自封 `<FeedbackButton>` | 全局反馈 |

## 二、图表（ECharts）

| 业务图表 | ECharts 类型 |
|---|---|
| 趋势 | line |
| 占比 | pie / donut |
| 排名/对比 | bar |
| 漏斗 | funnel |
| 留存矩阵 | heatmap |
| 路径 | sankey |
| 散点 / 矩阵 | scatter |
| 雷达 | radar |
| 地图 | geo |
| 词云 | wordCloud（扩展） |
| 仪表盘 | gauge |

封装为 `<BaseChart :type="..." :option="..." />` 统一组件。

## 三、自研业务组件（5 大核心）

### 1. `<KpiCard>`

```tsx
Props:
  title: string         // KPI 名称
  value: string|number  // 主数字
  prefix?: string       // ¥
  suffix?: string       // %
  trend?: 'up'|'down'|'flat'
  trendValue?: string   // "+12.3%"
  comparison?: string   // "vs 上周"
  status?: 'normal'|'warning'|'danger'
  loading?: boolean
  size?: 'sm'|'md'|'lg'
  onClick?: () => void
  
Events:
  @click

Slots:
  default - 自定义内容（覆盖默认布局）
```

**视觉规格**：
- 尺寸 sm/md/lg：高 80 / 120 / 160px
- 标题：14px / text-3
- 数字：32px / text-1 / 字重 600
- 趋势：14px，up=success/down=danger/flat=text-3
- 右上角：⚠ icon（status≠normal 时）
- hover: shadow-2 + cursor:pointer（onClick 时）

### 2. `<AiSuggestionCard>`

```tsx
Props:
  suggestion: {
    id: string
    title: string
    rationale: string[]
    expectedImpact: { metric: string, value: string }[]
    risks: string[]
    confidence: 1|2|3|4|5
  }
  actions?: ('review'|'analyze'|'record'|'reject')[]
  
Events:
  @action(type: string)
```

**视觉**：
- 宽 320-480px 自适应
- 高自适应，最大 600px
- 圆角 8px
- 阴影 shadow-1，hover shadow-2
- 顶部条背景：linear-gradient(135deg, #E6F4FF, #FFFFFF)
- 4 个按钮底部 footer

### 3. `<AnomalyCard>`

```tsx
Props:
  anomaly: {
    severity: 'P0'|'P1'|'P2'
    metric: string
    change: number
    owner: User
    rootCause: string
    suggestion: string
  }
  actions?: ('handle'|'transfer'|'chat')[]
```

**视觉**：
- P0 左侧 4px 红色边带
- P1 黄色
- P2 灰色
- 指标名 + 变化大字
- AI 归因摘要（≤ 2 行可展开）
- 建议动作（≤ 3 行）
- 操作按钮

### 4. `<DecisionCard>`

```tsx
Props:
  decision: {
    id: string
    title: string
    type: DecisionType
    decidedBy: User
    decidedAt: Date
    summary: string
    status: DecisionStatus
    reviewProgress?: { d30, d60, d90, d180 }
  }
```

**视觉**：
- 标题 + type tag
- 决策人 + 时间
- 摘要（≤ 3 行）
- 状态 Badge
- 复盘节点进度条
- [查看] [复盘] [分享]

### 5. `<AssistantBubble>`（决策助理浮窗）

```tsx
Props:
  context?: { type: string, id: string }
  defaultOpen?: boolean
  position?: 'bottom-right' (default) | 'bottom-left'

States:
  collapsed - 64×64 圆形按钮
  expanded - 500×600 对话框
  fullscreen - 右半屏

Slots:
  customQuickActions - 自定义快捷命令
```

**特性**：
- 默认右下角浮窗
- 点击展开为对话卡
- 可拖拽位置 + 调整宽度
- 顶部对话历史 + 底部输入框
- 输入框支持快捷命令：@ / # / 截图

## 四、其他业务组件

### `<UserProfile360>`

完整用户画像，详见 [11-embedded-components](../11-embedded-components/02-component-library.md)。

### `<SegmentBuilder>`

拖拽式分群构建，详见同上。

### `<JourneyDesigner>`

Journey 流程图编辑器，使用 X6 或自研。

### `<MetricSelector>`

指标选择器，支持搜索、分类、最近使用。

### `<ChannelBadge>`

触达通道标识：📱 站内 / 📧 邮件 / 💬 短信 / 🔔 Push / 💼 企微。

## 五、状态规范

| 状态 | 组件 | 用法 |
|---|---|---|
| 成功 | `message.success` | 右上 toast 3s |
| 警告 | `message.warning` | — |
| 错误 | `message.error` | — |
| 加载 | `Spin` / `Skeleton` | 优先骨架 |
| 确认 | `Modal.confirm` | 危险操作必须 |
| 引导 | `Tour` | 首次进入/新功能 |

## 六、组件治理

| 项 | 规范 |
|---|---|
| 组件库源代码 | monorepo（pnpm workspace） |
| 文档站 | VitePress + Storybook |
| 版本 | SemVer，至少 N-2 兼容 |
| 测试 | Vitest + Vue Test Utils ≥ 80% 覆盖 |
| 发布 | 公司私服 npm |
| 体积 | 单组件 gzip ≤ 30KB（不含 ECharts） |
| 主题 | CSS 变量化，支持业务系统覆盖 |
