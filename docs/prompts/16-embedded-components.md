# 提示词 16 · Vue3 嵌入式组件库

> 前置：`00-master-prompt.md`。  
> 参考：`docs/11-embedded-components/`。

---

## 提示词正文

```
你是前端组件库工程师。请开发 "@yourcompany/analytics-vue-components" 嵌入式组件库。

【目标】
供业务系统直接 import 嵌入分析能力，开箱即用，零摩擦接入。

【技术】

- Vue 3.4 peer dependency
- TypeScript 5（strict）
- Vite 5 Library Mode
- VitePress（文档站）+ Storybook（可选）
- Vitest + Vue Test Utils

【目录结构】

```
analytics-vue-components/
├── package.json
├── vite.config.ts
├── tsconfig.json
├── src/
│   ├── index.ts                          # 主入口
│   ├── setup.ts                          # 全局配置
│   ├── components/
│   │   ├── UserProfile360/
│   │   ├── EnterpriseProfile/
│   │   ├── FeatureHealthCard/
│   │   ├── QueryParamInsight/
│   │   ├── ChurnRiskCard/
│   │   ├── SegmentBuilder/
│   │   ├── SegmentPicker/
│   │   ├── JourneyDesigner/
│   │   ├── ABTestPanel/
│   │   ├── MetricCard/
│   │   ├── EventTimeline/
│   │   ├── RecommendBox/
│   │   ├── DecisionAssistant/
│   │   ├── GrowthOpportunity/
│   │   └── NPSCollector/
│   ├── composables/
│   │   ├── useAnalyticsClient.ts
│   │   ├── useAnalyticsTheme.ts
│   │   └── useAnalyticsAuth.ts
│   ├── shared/
│   │   ├── http.ts
│   │   ├── auth.ts
│   │   ├── error-boundary.ts
│   │   └── i18n.ts
│   ├── types/
│   └── styles/
│       └── tokens.css                    # CSS 变量
├── docs/                                  # VitePress
└── tests/
```

【全局配置 API】

```ts
// 业务方一次配置
import { setupAnalyticsComponents } from '@yourcompany/analytics-vue-components';

setupAnalyticsComponents({
  endpoint: 'https://api.analytics.yourcompany.com',
  getToken: async () => {
    return await fetch('/api/get-analytics-token').then(r => r.text());
  },
  theme: {
    primaryColor: '#1677FF',
    // 任何 CSS 变量
  },
  locale: 'zh-CN',
  errorBoundary: true,
  fallbackComponent: CustomFallback,
  debug: false,
});
```

【15 个组件详细规约】

每个组件必须：
1. Props 强类型 + 必填校验
2. Emits 完整定义
3. 内置数据请求（业务方只传 ID）
4. 权限自动校验
5. Loading / Error / Empty / NoPermission 状态
6. CSS 变量化（支持主题覆盖）
7. Tree-shaking 友好
8. 单组件 gzip ≤ 30KB
9. 完整 d.ts
10. 单测覆盖 ≥ 80%

### UserProfile360（详细）

```vue
<template>
  <ErrorBoundary :fallback="fallbackComponent">
    <a-spin :spinning="loading">
      <div class="up360-container">
        <UserBasicInfo :user="user" />
        <UserHealthMetrics :metrics="metrics" />
        <UserLifecycleStage :stage="lifecycle" />
        <UserBehaviorTimeline :events="timeline" />
        <UserCommonFeatures :features="features" />
        <UserUnusedCoreFeatures :features="unusedCore" />
        <UserPurchaseHistory :purchases="purchases" />
        <UserReachHistory :reaches="reaches" />
        <UserSessionReplayEntry />
        <UserOneClickActions @action="handleAction" />
      </div>
    </a-spin>
  </ErrorBoundary>
</template>

<script setup lang="ts">
interface Props {
  userId: string;
  orgContext?: string;
  layout?: 'compact' | 'full';
  showActions?: boolean;
  include?: ('basic'|'health'|'lifecycle'|'timeline'|'features'|'purchases'|'reaches'|'actions')[];
}

interface Emits {
  (e: 'profile-loaded', user: User): void;
  (e: 'action', type: string, payload: unknown): void;
  (e: 'error', error: Error): void;
}

const props = withDefaults(defineProps<Props>(), {
  layout: 'full',
  showActions: true,
  include: () => ['basic', 'health', 'lifecycle', 'timeline', 'features', 'purchases', 'reaches', 'actions']
});

const emit = defineEmits<Emits>();

const { user, metrics, lifecycle, timeline, ... } = useUserProfile360(props.userId);
</script>
```

### 其他组件（按 docs/11-embedded-components/02-component-library.md 详细规约实现）

【4 种嵌入形态】

1. **NPM 组件**（默认）：业务方 import 使用
2. **Web Components**（v2）：基于 Lit + Vue3 包装，跨框架
3. **微前端**（wujie）：作为子应用，整页嵌入
4. **iframe**（兜底）：提供独立页面 URL

【SSO 打通】

- 业务后端签发短期 Token（≤ 15min）
- 组件内 useAnalyticsAuth() 自动获取并刷新
- Token 接近过期自动刷新

【主题系统】

- 所有样式用 CSS 变量
- setupAnalyticsComponents 可覆盖
- 支持 light / dark
- 业务系统主题色自动适配

【国际化】

- vue-i18n + 内置 zh-CN / en-US
- 业务方可注入自定义翻译

【错误边界】

- 全局 ErrorBoundary 包装
- 组件级降级
- 网络失败 → "暂不可用 + 重试"
- 数据为空 → Empty
- 无权限 → 友好提示 + 联系方式
- 绝不白屏

【文档站（VitePress）】

每个组件文档：
- 描述
- Props 表
- Emits 表
- Slots
- 示例（live demo）
- API
- 设计原则
- 注意事项

【发布】

- 公司私服 npm（Verdaccio / Nexus）
- SemVer
- N-2 兼容
- CHANGELOG.md

【单元测试】

每个组件：
- 默认渲染
- Props 校验
- Loading / Error / Empty 状态
- 用户交互
- 事件 emit
- 权限校验

【验收】

1. 15 个组件全部实现
2. 单组件体积达标（≤ 30KB）
3. VitePress 文档站可访问
4. 在干净 Vue3 项目可 import 使用
5. 主题覆盖生效
6. 错误边界生效
7. 单测覆盖 ≥ 80%
8. 发布到私服成功

【交付物】

- 完整组件库源码
- VitePress 文档站
- 集成示例工程
- CHANGELOG.md
- 私服发布脚本

【禁止】

- 禁止依赖 UI 库（如 Ant Design Vue，组件库要自包含）
- 禁止内联样式（动态除外）
- 禁止 any
- 禁止白屏
- 禁止 AppSecret 出现

现在请生成。
```
