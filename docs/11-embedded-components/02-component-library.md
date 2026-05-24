# Vue3 嵌入组件库（@yourcompany/analytics-vue-components）

## 一、组件清单（V1 必须 15 个）

| 组件 | 用途 | 集成方 |
|---|---|---|
| `<UserProfile360 userId />` | 用户 360° 画像 | CRM、客服后台 |
| `<EnterpriseProfile orgId />` | 企业 360° 画像 | CSM 工作台 |
| `<FeatureHealthCard featureCode />` | 单功能健康度 | PM 后台 |
| `<QueryParamInsight scope />` | 查询参数分析 | 业务后台 |
| `<ChurnRiskCard userId />` | 流失风险 | CSM、销售 |
| `<SegmentBuilder />` | 自助分群 | 运营平台 |
| `<SegmentPicker />` | 分群选择器（轻量） | 触达配置 |
| `<JourneyDesigner />` | 旅程编排 | 运营平台 |
| `<ABTestPanel />` | AB 实验入口 | PM 后台、运营 |
| `<MetricCard metric />` | 单指标卡 | 任何看板 |
| `<EventTimeline userId />` | 行为时间线 | 客服后台 |
| `<RecommendBox type />` | 推荐组件 | 平台、CRM |
| `<DecisionAssistant context />` | 对话助理 | 全系统 |
| `<GrowthOpportunity />` | 增长机会卡 | 运营/销售 |
| `<NPSCollector />` | NPS 收集器 | 业务系统任意页 |

## 二、组件设计原则

1. **强类型 Props + 主题变量**
2. **数据 + UI 一体**：组件自请求数据，业务方只传 ID
3. **权限内置**：组件自动校验
4. **事件外抛**：emit 事件供业务方监听
5. **Tree-shaking + 按需引入**：单组件 gzip ≤ 30KB
6. **统一 Token 鉴权**：一次配置全局生效
7. **降级方案**：网络失败显示"暂不可用 + 重试"，绝不白屏

## 三、4 种嵌入形态

| 形态 | 工具 | 适用 | 优点 | 注意 |
|---|---|---|---|---|
| **NPM 组件** | Vue3 组件库 | 单组件嵌入 Vue3 项目 | 灵活、轻量 | 需要代码改动 |
| **Web Components** | Lit + Vue3 包装 | 跨框架（Vue2/React/Angular） | 隔离好 | 调试稍复杂 |
| **微前端整页** | wujie | 整模块嵌入 | 改动最少 | 需主框架支持 |
| **iframe** | 原生 | 兜底兼容 | 0 改动 | 样式/通信受限 |

## 四、详细组件规约（核心 5 个）

### `<UserProfile360 />`

```ts
Props:
  userId: string                  // 必填
  orgContext?: string             // 可选，企业上下文
  layout?: 'compact' | 'full'     // 默认 full
  showActions?: boolean           // 默认 true
  onAction?: (action: string, payload: any) => void

Emits:
  @profile-loaded
  @action(type, payload)
  @error
```

模块包含：
- 基础信息（名字 / 企业 / 角色 / 会员等级）
- 健康分卡片 + 流失风险 + LTV
- 行为时间线（最近 30 天）
- 常用功能 TOP10 + 未使用核心功能
- 订单/合同/付费历史
- 触达记录
- Session Replay 入口
- 一键运营动作（打标签 / 加入分群 / 触发触达）

### `<EnterpriseProfile orgId />`

类似 UserProfile360，但维度升级为企业：
- 企业基础 + 套餐 + 续费日 + CSM 负责人
- Seat 激活率、模块开通率、活跃员工数
- Admin vs 普通成员行为
- 异常信号

### `<FeatureHealthCard featureCode />`

模块包含：
- 五维评分雷达
- 使用漏斗
- 参数使用分析
- 结果分析
- 二次查询率
- 报错与求助
- 设备/角色/版本分布
- Session Replay
- 用户反馈

### `<SegmentBuilder />`

```ts
Props:
  initialRules?: SegmentRule[]
  onSave?: (segment) => void
  onPreview?: (count) => void
```

- 拖拽式条件配置
- 实时人数预估
- 一键导出 / 触达 / 启动实验

### `<JourneyDesigner />`

```ts
Props:
  initialJourney?: Journey
  onSave?: (journey) => void
```

- 流程图编辑器（基于 X6）
- 节点类型：触发 / 条件 / 等待 / 触达 / AB
- 实时效果追踪面板

## 五、统一接入配置

```ts
// 全局配置（业务方一次性）
import { setupAnalyticsComponents } from '@yourcompany/analytics-vue-components';

setupAnalyticsComponents({
  endpoint: 'https://api.analytics.yourcompany.com',
  getToken: async () => {
    return await fetch('/api/get-analytics-token').then(r => r.text());
  },
  theme: {
    primaryColor: '#1677FF',
    // 覆盖任何 CSS 变量
  },
  locale: 'zh-CN',
  errorBoundary: true,
  fallbackComponent: CustomFallback,
});
```

## 六、组件治理

| 项 | 规范 |
|---|---|
| 仓库 | monorepo（pnpm workspace） |
| 文档站 | VitePress + Storybook（每组件 Demo + Props + 事件） |
| 版本 | SemVer，至少 N-2 兼容 |
| 测试 | Vitest + Vue Test Utils ≥ 80% 覆盖 |
| 发布 | 公司私服 npm（Verdaccio / Nexus） |
| 体积 | 单组件 gzip ≤ 30KB（不含 ECharts） |
| 主题 | CSS 变量化 |
| 使用统计 | 每组件埋点，识别"高使用 vs 鸡肋" |

## 七、降级与错误边界

```ts
// 全局错误边界包装
<ErrorBoundary :fallback="DefaultFallback">
  <UserProfile360 :userId="userId" />
</ErrorBoundary>

// 网络失败 → 显示"暂不可用 + 重试"
// 数据为空 → 显示 Empty 状态
// 无权限 → 显示 "您暂无权限" + 联系方式
// 加载中 → Skeleton
```

## 八、SSO 打通方案

```
业务系统主框架已登录
   ↓
调用 /api/get-analytics-token 获取短期 Token（≤ 15min）
   ↓
组件初始化时调用中台 /openapi/v1/auth/exchange
   ↓
换取中台内部 JWT（无感登录）
   ↓
后续请求自动带 JWT
   ↓
Token 接近过期 → 自动刷新
```
