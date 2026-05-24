# 提示词 04 · Vue3 数据上报 SDK

> 前置：`00-master-prompt.md`。参考：`docs/11-embedded-components/01-sdk-design.md`。

---

## 提示词正文

```
你是一名前端 SDK 开发专家。请开发 "@yourcompany/analytics-sdk"（Vue3 + TypeScript）。

【目标】
给业务前端提供"一行代码接入、自动+手动埋点、断网续传"的埋点 SDK。

【技术要求】

- 框架：Vue 3.4 peer dependency
- 语言：TypeScript 5（strict）
- 构建：Vite 5 Library Mode
- 输出：ESM + UMD + d.ts 三产物
- 体积：gzip ≤ 30KB
- 不允许依赖：lodash / moment / axios / 任何 UI 库

【目录结构】

```
analytics-sdk/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── src/
│   ├── index.ts                 # 主入口
│   ├── plugin.ts                # Vue 插件
│   ├── client.ts                # 核心客户端
│   ├── transport/
│   │   ├── beacon.ts
│   │   ├── fetch.ts
│   │   └── xhr.ts
│   ├── auto-track/
│   │   ├── pv.ts
│   │   ├── error.ts
│   │   ├── performance.ts       # Web Vitals
│   │   └── stay-time.ts
│   ├── storage/
│   │   └── indexed-db.ts        # 离线缓存
│   ├── directives/
│   │   └── v-track.ts
│   ├── composables/
│   │   └── use-analytics.ts
│   ├── types/
│   │   └── index.ts
│   └── utils/
│       ├── hash.ts
│       ├── debounce.ts
│       └── env.ts
├── tests/
└── README.md
```

【核心 API】

```ts
// 插件配置
export interface AnalyticsOptions {
  endpoint: string;
  appKey: string;
  getToken: () => Promise<string>;
  autoTrack?: {
    pv?: boolean;          // 默认 true
    click?: boolean;       // 默认 false
    error?: boolean;       // 默认 true
    performance?: boolean; // 默认 true
    stayTime?: boolean;    // 默认 true
  };
  batch?: {
    size?: number;             // 默认 50
    intervalMs?: number;       // 默认 5000
  };
  storage?: {
    maxOfflineEvents?: number; // 默认 1000
  };
  debug?: boolean;
}

// 插件安装
import { AnalyticsPlugin } from '@yourcompany/analytics-sdk';
app.use(AnalyticsPlugin, options);

// 指令
<button v-track="'event_code'">
<button v-track="{ event: 'xxx', props: { foo: 'bar' } }">

// Composable
const { track, identify, setUser, setSuperProps, flush } = useAnalytics();

// 客户端 API
export interface AnalyticsClient {
  track(event: string, props?: Record<string, unknown>): void;
  identify(userId: string, traits?: Record<string, unknown>): void;
  setUser(properties: Record<string, unknown>): void;
  setSuperProps(properties: Record<string, unknown>): void;
  flush(): Promise<void>;
}
```

【自动采集能力】

1. **PV**（路由切换触发）
   - Vue Router 钩子集成
   - track 'page_view' 含 url / referrer / 停留时长

2. **Web Vitals**（使用 web-vitals lib 或自实现）
   - LCP / FID / CLS / INP / FCP / TTFB
   - 页面卸载时上报

3. **JS 错误**
   - window.onerror
   - window.unhandledrejection
   - 资源加载错误（error event capture）

4. **停留时长**
   - visibilitychange 监听
   - 页面隐藏时上报真实停留时长

5. **点击（可选）**
   - 启用后自动 track 所有点击
   - 含元素 selector / text / xpath

【上报通道优先级】

```
1. navigator.sendBeacon (首选)
2. fetch(keepalive: true)
3. XMLHttpRequest
```

【离线缓存】

- 基于 idb-keyval（轻量封装 IndexedDB）或自实现
- 失败/离线 → 入 IndexedDB
- 最多 1000 条，FIFO 丢弃
- 网络恢复 + 下次启动自动重发
- 重发批量（不一次性全发）

【批量上报】

- 队列：内存 + IndexedDB 双重
- 触发：满 batch.size 或定时 intervalMs
- 页面卸载（pagehide / beforeunload）强制 flush（sendBeacon）

【鉴权】

- 业务后端签发短期 Token（≤ 15min）
- SDK 通过 options.getToken 获取
- Token 接近过期自动刷新（提前 1min）
- AppSecret 绝不出现在前端

【性能要求】

- 主线程影响 < 10ms / track 调用
- 单次上报 < 50ms（不阻塞）
- 离线缓存读写 < 100ms

【兼容性】

- 浏览器：Chrome 90+ / Edge 90+ / Safari 14+ / Firefox 88+
- SSR 兼容（typeof window 检查）
- 小程序兼容（适配层，可选）

【测试】

- Vitest + jsdom
- 覆盖率 ≥ 80%
- 必测：
  - 插件安装
  - track / identify / setUser
  - 自动 PV / 错误 / 性能
  - 离线缓存重发
  - 批量触发
  - sendBeacon 降级
  - 多 tab 不冲突

【交付物】

- 完整源代码 + tests
- vitepress 文档站（含 Demo）
- CHANGELOG.md
- 接入示例：Vue3 + Vite 项目接入演示
- 性能报告（bundlephobia 截图）
- 发布到公司私服 npm

【禁止】

- 禁止 document.write
- 禁止引用大包（lodash / moment / axios）
- 禁止主线程同步阻塞
- 禁止 any（除非显式注释原因）
- 禁止 console.log 上线代码（debug 模式除外）

【安全】

- AppSecret 绝不出现
- Token 自动失效后丢弃
- 上报数据敏感字段（手机/身份证）自动脱敏
- 同源策略 + CORS 严格

现在请生成完整代码 + 测试 + 文档。
```
