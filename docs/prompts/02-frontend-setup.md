# 提示词 02 · 前端 Vue3 项目初始化

> 前置：先阅读 `00-master-prompt.md`。

---

## 提示词正文

```
你是一名资深前端架构师。请基于以下规约创建 "analytics-platform-frontend" 项目。

【目标】
生成可直接 `pnpm i && pnpm dev` 运行、`pnpm build` 通过的前端项目骨架。
本项目同时作为"独立管理后台" + "wujie 微前端子应用参考实现"。

【技术栈】

- Vue 3.4+
- TypeScript 5+（strict mode）
- Vite 5+
- pnpm 8+
- Pinia 2+（含持久化）
- Vue Router 4+
- Ant Design Vue 4.x
- ECharts 5+
- @vueuse/core
- axios + 自封装请求
- vue-i18n 9+
- wujie-vue3（微前端）
- Vitest + Vue Test Utils + @testing-library/vue
- Playwright（E2E）
- ESLint + Prettier + Stylelint + Husky + lint-staged + commitlint

【目录结构】

```
analytics-platform-frontend/
├── package.json
├── pnpm-workspace.yaml          # 可选 monorepo
├── vite.config.ts
├── tsconfig.json
├── .eslintrc.cjs
├── .prettierrc.json
├── .stylelintrc.cjs
├── .editorconfig
├── .gitignore
├── .husky/
│   ├── pre-commit
│   └── commit-msg
├── commitlint.config.cjs
├── public/
├── src/
│   ├── main.ts
│   ├── App.vue
│   ├── router/
│   │   ├── index.ts
│   │   └── routes/
│   ├── stores/                   # Pinia 全局 store
│   │   ├── index.ts
│   │   ├── user.ts
│   │   └── app.ts
│   ├── features/                 # 按业务域组织
│   │   ├── dashboard-s1/
│   │   │   ├── components/
│   │   │   ├── composables/
│   │   │   ├── stores/
│   │   │   ├── types/
│   │   │   ├── services/
│   │   │   └── views/
│   │   ├── dashboard-s2/
│   │   ├── ... 其他看板
│   │   ├── segment/
│   │   ├── journey/
│   │   ├── decision/
│   │   └── assistant/
│   ├── shared/                   # 共享
│   │   ├── components/           # 业务通用组件
│   │   │   ├── KpiCard.vue
│   │   │   ├── AiSuggestionCard.vue
│   │   │   ├── AnomalyCard.vue
│   │   │   ├── DecisionCard.vue
│   │   │   ├── AssistantBubble.vue
│   │   │   └── BaseChart.vue
│   │   ├── composables/
│   │   ├── directives/           # v-permission, v-track
│   │   ├── utils/
│   │   ├── http/                 # axios 封装
│   │   │   ├── request.ts
│   │   │   ├── interceptors.ts
│   │   │   └── types.ts
│   │   └── constants/
│   ├── layouts/
│   │   ├── DefaultLayout.vue
│   │   └── EmbedLayout.vue       # 微前端嵌入用
│   ├── locales/
│   │   ├── zh-CN.json
│   │   └── en-US.json
│   ├── styles/
│   │   ├── tokens.css            # 设计 Tokens
│   │   ├── reset.css
│   │   └── global.css
│   └── types/
│       └── global.d.ts
├── tests/
│   ├── unit/
│   └── e2e/
├── Dockerfile
├── nginx.conf
└── README.md
```

【vite.config.ts 关键配置】

- @ alias 指向 src
- 自动导入（unplugin-auto-import、unplugin-vue-components 按需引入 Ant Design Vue）
- 环境变量（VITE_ 前缀）
- 代理后端 API
- 构建分包（vendor/echarts/antd 独立 chunk）
- 同时打包两个入口：
  - 默认：独立应用
  - 嵌入：wujie 子应用入口

【tsconfig.json strict 配置】

- strict: true
- noUnusedLocals: true
- noUnusedParameters: true
- noImplicitAny: true
- noImplicitReturns: true
- strictNullChecks: true
- 路径别名

【shared/http/request.ts 核心实现】

- axios 实例
- 拦截器：
  - 请求：注入 JWT、traceId、Idempotency-Key、loading
  - 响应：统一 Result<T> 解包，错误码处理
- 401 → 跳转登录
- 403 → 跳转无权限页
- 429 → 限流提示 + Retry-After
- 5xx → 友好错误 + Sentry 上报
- 取消 token

【全局指令】

- v-permission="permission-code"  权限控制
- v-track="'event-code'"           埋点
- v-debounce-click                 防抖点击
- v-copy                           复制

【路由配置】

- 路由懒加载
- 路由 meta：title / requiresAuth / permissions
- 404 / 403 / 500 错误页
- 路由守卫：登录检查 + 权限检查 + 埋点

【ESLint 规则】

- @vue/eslint-config-typescript/recommended
- @vue/eslint-config-prettier
- 自定义规则：禁止 any（warning） / 禁止 console（warning） / 强制 explicit-function-return-type

【Husky + lint-staged】

```
pre-commit: lint-staged
  - *.{js,ts,vue,jsx,tsx}: eslint --fix
  - *.{css,less,scss,vue}: stylelint --fix
  - *.{js,ts,vue,json,md}: prettier --write
commit-msg: commitlint
```

【commitlint.config.cjs】

- @commitlint/config-conventional
- 中文 type 描述
- type 限制：feat/fix/docs/style/refactor/perf/test/build/ci/chore/revert

【tests 配置】

- Vitest + jsdom
- 覆盖率门禁 60%（核心 80%）
- Playwright 配置 chromium + webkit + firefox
- E2E 测试关键路径

【Dockerfile】

```
# 多阶段构建
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

【nginx.conf 关键配置】

- gzip + brotli
- SPA history mode (try_files)
- 静态资源缓存
- 安全 Headers (CSP, X-Frame-Options, X-Content-Type-Options)
- API 反代后端

【验收标准】

1. `pnpm install` + `pnpm dev` 启动成功
2. `pnpm build` 构建通过，产物 < 5MB
3. `pnpm test:unit` 通过
4. `pnpm lint` 0 错误
5. `pnpm format` 通过
6. 默认页面可见欢迎页 + 路由跳转
7. Mock 数据下能完整跑通登录 → 看板
8. Dockerfile 可构建镜像 < 50MB
9. 浏览器兼容：Chrome / Edge / Safari 最新两版
10. 无障碍：WCAG AA

【输出格式】

按文件组织，每个文件用 ```{语言}:{完整路径}``` 包裹。
按目录顺序输出，最后输出 README.md。

【禁止】

- 禁止 any（除非 // eslint-disable-next-line + 注释原因）
- 禁止内联样式（除非动态）
- 禁止组件内直接 axios
- 禁止 console.log 上线代码
- 禁止 lodash 全包（按需 import 单函数）
- 禁止 moment（用 dayjs）

现在请开始生成。
```
