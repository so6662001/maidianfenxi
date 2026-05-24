# 对外集成模式

## 一、三种集成方式总览

| 方式 | 形态 | 适用场景 | 接入成本 |
|---|---|---|---|
| **① 数据上报 SDK** | Java SDK / Vue3 SDK | 业务系统埋点 | 中（一次性，但需埋点设计） |
| **② 业务消费 API** | RESTful + OpenAPI | 业务系统查指标、拉标签、触发分群 | 中 |
| **③ 嵌入式 SDK** | Vue3 组件 / 微前端 | 业务系统直接嵌入分析能力 | 低（最快接入方式） |

## 二、方式 ① · 数据上报 SDK

### Java SDK 设计要点

```yaml
artifactId: analytics-java-sdk
groupId: com.yourcompany.analytics
依赖最小: OkHttp + Jackson + SLF4J（不依赖 Spring）

提供两个产物:
  - analytics-java-sdk           # 核心 SDK，任意 Java 8+ 项目可用
  - analytics-spring-boot-starter # Spring Boot 自动装配
```

**关键能力**：
- 异步批量上报，不阻塞业务线程
- 失败本地落盘 + 断点续传（Chronicle Queue）
- HMAC 签名鉴权
- OpenTelemetry traceId 透传
- AOP 注解 `@Track("event_code")` 自动埋点

**业务方接入示例**：
```java
// Spring Boot 项目
application.yml:
  analytics:
    endpoint: https://collector.xxx.com
    app-key: xxx
    app-secret: xxx

// 启动类
@EnableAnalytics

// 使用
@Autowired AnalyticsClient client;
client.track(userId, "invoice_export", Map.of("count", 100));

// 或注解
@Track(event="invoice_export", props="#{result.count}")
public ExportResult exportInvoice(...) {...}
```

### Vue3 SDK 设计要点

```yaml
package: "@yourcompany/analytics-sdk"
体积: gzip ≤ 30KB
依赖: 仅 peer dependency vue@^3.4
```

**关键能力**：
- 自动采集：PV / UV / Web Vitals / JS Error
- 指令埋点：`v-track="'event_code'"`
- Composable：`useAnalytics()`
- 离线缓存：IndexedDB
- 短期 Token 鉴权（不存 Secret）

**业务方接入示例**：
```ts
// main.ts
import { AnalyticsPlugin } from '@yourcompany/analytics-sdk';
app.use(AnalyticsPlugin, {
  endpoint: 'https://collector.xxx.com',
  appKey: 'xxx',
  getToken: () => fetch('/api/get-analytics-token').then(r => r.text()),
  autoTrack: { pv: true, error: true, performance: true }
});

// 组件中
<button v-track="'invoice_export_click'">导出</button>

// 或手动
const { track } = useAnalytics();
track('custom_event', { foo: 'bar' });
```

## 三、方式 ② · 业务消费 API

### API 规范

| 项 | 规范 |
|---|---|
| 路径 | `/openapi/v{version}/...` |
| 版本 | URL 路径版本号，老版本至少保留 6 个月 |
| 鉴权 | AppKey + AppSecret + HMAC-SHA256 + 时间戳防重放 |
| 限流 | Sentinel，按 AppKey + 接口 QPS/分钟配额 |
| 幂等 | 写接口必须 `X-Idempotency-Key` |
| 分页 | 游标分页 cursor + limit，max 200 |
| 响应 | `{code, message, data, traceId, requestId}` |
| 错误码 | 6 位数字（前 2 服务码 + 后 4 业务码） |
| 文档 | 自动生成 OpenAPI 3.0 |
| SDK | 自动生成 Java/TypeScript/Python Client |

### 核心 API 清单（V1）

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/metric/query` | POST | 指标查询（DSL） |
| `/openapi/v1/user/profile/{userId}` | GET | 用户画像 |
| `/openapi/v1/org/profile/{orgId}` | GET | 企业画像 |
| `/openapi/v1/segment/{segmentId}/users` | GET | 分群用户列表 |
| `/openapi/v1/segment/preview` | POST | 分群人数预估 |
| `/openapi/v1/tag/user/{userId}` | GET | 用户标签 |
| `/openapi/v1/event/track` | POST | 单事件上报（备用，主推 SDK） |
| `/openapi/v1/journey/trigger` | POST | 触发 Journey |
| `/openapi/v1/experiment/{expId}/variant` | GET | 用户实验分组 |
| `/openapi/v1/insight/anomaly` | GET | 最近异动列表 |

### 业务方调用示例

```bash
# 计算签名
timestamp=$(date +%s)
nonce=$(uuidgen)
sign_string="POST\n/openapi/v1/metric/query\n${timestamp}\n${nonce}\n${body_md5}"
signature=$(echo -n "$sign_string" | openssl dgst -sha256 -hmac "$APP_SECRET" -binary | base64)

curl -X POST https://api.xxx.com/openapi/v1/metric/query \
  -H "X-App-Key: $APP_KEY" \
  -H "X-Timestamp: $timestamp" \
  -H "X-Nonce: $nonce" \
  -H "X-Signature: $signature" \
  -H "X-Idempotency-Key: $request_id" \
  -d '{"metric":"saas_dau","dimensions":["product"],"dateRange":{"start":"2026-05-01","end":"2026-05-23"}}'
```

## 四、方式 ③ · 嵌入式 SDK

### 4 种嵌入形态

| 形态 | 工具 | 适用 | 优点 | 注意 |
|---|---|---|---|---|
| **NPM 组件** | Vue3 组件库 | 单组件嵌入 Vue3 项目 | 灵活、轻量 | 需要代码改动 |
| **Web Components** | Lit + Vue3 包装 | 跨框架（Vue2/React/Angular） | 隔离好 | 调试稍复杂 |
| **微前端整页** | wujie | 整模块嵌入 | 改动最少 | 需要主框架支持 |
| **iframe** | 原生 | 兜底兼容 | 0 改动 | 样式/通信受限 |

### NPM 组件库

```yaml
package: "@yourcompany/analytics-vue-components"

组件清单（V1）:
  - <UserProfile360 userId="..." />      # 用户画像
  - <EnterpriseProfile orgId="..." />    # 企业画像
  - <FeatureHealthCard featureCode="..." /> # 功能健康
  - <QueryParamInsight scope="..." />    # 查询参数分析
  - <ChurnRiskCard userId="..." />       # 流失风险
  - <SegmentBuilder />                   # 自助分群
  - <SegmentPicker />                    # 分群选择
  - <JourneyDesigner />                  # 旅程编排
  - <ABTestPanel />                      # AB 实验入口
  - <MetricCard metric="..." />          # 单指标卡
  - <EventTimeline userId="..." />       # 行为时间线
  - <RecommendBox type="..." />          # 推荐组件
  - <DecisionAssistant context="..." /> # 对话助理
  - <GrowthOpportunity />                # 增长机会
  - <NPSCollector />                     # NPS 收集器
```

### 嵌入式组件设计原则

1. **强类型 Props + 主题变量**
2. **数据 + UI 一体**：组件自己请求数据，业务方只传 ID
3. **权限内置**：组件自动校验用户权限
4. **事件外抛**：emit 事件供业务方监听
5. **Tree-shaking + 按需引入**：单组件 gzip ≤ 30KB
6. **统一 Token 鉴权**：一次配置全局生效
7. **降级方案**：网络失败显示"暂不可用 + 重试"，绝不白屏

### 微前端集成（wujie）

```ts
// 业务系统主框架
import WujieVue from 'wujie-vue3';

<WujieVue
  width="100%"
  height="100%"
  name="analytics-segment-page"
  url="https://analytics.xxx.com/embedded/segment"
  :props="{ token: userToken, theme: 'light' }"
  @on-message="handleMessage"
/>

// 中台子应用
window.$wujie?.bus.$on('theme-change', (theme) => { ... });
```

### SSO 打通方案

```
业务系统主框架已登录
   ↓
调用 /api/get-analytics-token 获取短期 Token（15min）
   ↓
通过 wujie props 传给子应用
   ↓
子应用初始化时调用中台 /openapi/v1/auth/exchange
   ↓
换取中台内部 JWT（无感登录）
```

## 五、接入流程（业务方视角）

```
1. 业务方申请接入
   ↓
2. 中台分配 AppKey + AppSecret + 接入文档
   ↓
3. 业务方选择接入方式（① / ② / ③ 可组合）
   ↓
4. 中台提供 demo 工程 + 接入大使支持
   ↓
5. 业务方开发联调（沙箱环境）
   ↓
6. 中台验收：埋点覆盖率、调用规范、错误率
   ↓
7. 切到 prod 环境
   ↓
8. 持续运营：使用统计、培训、优化
```

## 六、SLA 与支持

| 项 | 标准 |
|---|---|
| 接入响应 | T+1 工作日 |
| 平均接入工时 | 目标 ≤ 3 个工作日 |
| 沙箱可用性 | 7×24，与 prod 数据隔离 |
| 接入大使支持 | 企微群驻点答疑 |
| 文档完整度 | 100% 接口含示例 |
| SDK 兼容期 | 至少 N-2 版本 |
