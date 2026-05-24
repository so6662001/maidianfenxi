# SDK 设计（Java + Vue3）

## 一、Java SDK 设计

### 包结构

```
分两个产物：
- analytics-java-sdk            核心 SDK（不依赖 Spring）
- analytics-spring-boot-starter Spring Boot 自动装配
```

### 包名 & 坐标

```xml
<groupId>com.yourcompany.analytics</groupId>
<artifactId>analytics-java-sdk</artifactId>
<version>1.0.0</version>

<groupId>com.yourcompany.analytics</groupId>
<artifactId>analytics-spring-boot-starter</artifactId>
<version>1.0.0</version>
```

### 依赖最小化

```
仅依赖：
- okhttp 4.x
- jackson-databind 2.x
- slf4j-api 2.x
- chronicle-queue（可选，本地落盘）
```

**禁止依赖**：
- Spring（避免与业务方版本冲突）
- Guava / Apache Commons（业务方可能用不同版本）

### 核心 API

```java
public interface AnalyticsClient {
    // 事件上报
    void track(String userId, String eventCode, Map<String, Object> properties);
    void track(String userId, String eventCode, Map<String, Object> properties, Long eventTime);
    
    // 身份关联
    void identify(String anonymousId, String userId, Map<String, Object> traits);
    void groupAssoc(String userId, String orgId);
    
    // 用户属性
    void setUserProperty(String userId, Map<String, Object> properties);
    void setUserOnce(String userId, Map<String, Object> properties);
    
    // 高级
    void flush();      // 立即发送
    void close();      // 关闭客户端
}
```

### Spring Boot Starter

```yaml
# application.yml
analytics:
  enabled: true
  endpoint: https://collector.yourcompany.com
  app-key: ${ANALYTICS_APP_KEY}
  app-secret: ${ANALYTICS_APP_SECRET}
  batch-size: 200
  flush-interval-ms: 2000
  max-queue-size: 100000
  enable-disk-buffer: true
  disk-buffer-path: ${java.io.tmpdir}/analytics-buffer
  http-timeout-ms: 5000
  http-max-retries: 3
  debug: false
```

```java
// 启动类
@SpringBootApplication
@EnableAnalytics
public class App { ... }

// 注入使用
@Service
public class InvoiceService {
    @Autowired AnalyticsClient analytics;
    
    public void exportInvoice(String userId, ExportRequest req) {
        // 业务逻辑
        analytics.track(userId, "saas_invoice_export", Map.of(
            "count", req.getCount(),
            "format", req.getFormat()
        ));
    }
}
```

### AOP 注解埋点

```java
@Component
public class OrderService {
    @Track(event = "order_create", props = "#{result.id}")
    public Order createOrder(OrderRequest req) {
        // ...
    }
}
```

### 异步批量上报机制

```
业务调用 track()
   ↓
入 BlockingQueue（默认 100k 容量）
   ↓
后台 Consumer 线程（单线程，避免乱序）
   ↓
满 batch_size 或 flush_interval 触发
   ↓
HTTP POST 到 Collector（带 HMAC 签名）
   ↓
成功：清队列
失败：指数退避 3 次重试
   ↓
仍失败：写本地磁盘队列（Chronicle Queue）
   ↓
下次启动时重新加载并上报
```

### 鉴权

```
HMAC-SHA256 签名：
sign_string = METHOD + "\n" + PATH + "\n" + TIMESTAMP + "\n" + NONCE + "\n" + BODY_MD5
signature = HMAC-SHA256(sign_string, APP_SECRET)
Base64 编码

Headers:
  X-App-Key: ${APP_KEY}
  X-Timestamp: ${TIMESTAMP}
  X-Nonce: ${UUID}
  X-Signature: ${signature}
```

### 优雅关闭

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    analyticsClient.flush();  // 强制 flush 队列
    analyticsClient.close();
}));
```

### OpenTelemetry 集成

自动透传 traceId 到 Collector，与 SkyWalking 链路打通。

### 安全要求

- AppSecret 仅在配置中，不参与日志
- 失败重试间隔 ≥ 1s（避免风暴）
- 队列满时按策略丢弃（最旧/最新可配置）
- 超大 properties（> 64KB）自动截断 + 警告

## 二、Vue3 SDK 设计

### 包名

```json
{
  "name": "@yourcompany/analytics-sdk",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.cjs.js",
  "module": "./dist/index.es.js",
  "types": "./dist/index.d.ts",
  "peerDependencies": {
    "vue": "^3.4.0"
  }
}
```

### 体积控制

- ESM + UMD + d.ts 三产物
- gzip ≤ 30KB
- 仅依赖 Vue 3.4+
- 禁止引入 lodash / moment / axios 等大包
- 使用原生 fetch / URLSearchParams

### 安装方式

```ts
// main.ts
import { createApp } from 'vue';
import { AnalyticsPlugin } from '@yourcompany/analytics-sdk';

app.use(AnalyticsPlugin, {
  endpoint: 'https://collector.yourcompany.com',
  appKey: import.meta.env.VITE_ANALYTICS_APP_KEY,
  getToken: async () => {
    // 业务后端签发短期 Token（不存 Secret）
    const res = await fetch('/api/get-analytics-token');
    return res.text();
  },
  autoTrack: {
    pv: true,
    error: true,
    performance: true,
    click: false  // 默认关闭，按需开
  },
  debug: import.meta.env.DEV
});
```

### 指令埋点

```vue
<template>
  <button v-track="'invoice_export_click'">导出</button>
  <button v-track="{ event: 'custom', props: { foo: 'bar' } }">自定义</button>
</template>
```

### Composable

```ts
import { useAnalytics } from '@yourcompany/analytics-sdk';

const { track, identify, setUser, setSuperProps } = useAnalytics();

track('event_code', { key: 'value' });
identify('userId');
setUser({ name: 'Alice', role: 'admin' });
setSuperProps({ product: 'saas' });  // 所有后续事件自动带
```

### 自动采集能力

| 能力 | 默认 | 说明 |
|---|---|---|
| PV | 开 | Vue Router 钩子 |
| 路由切换 | 开 | 自动 track 'page_view' |
| Web Vitals | 开 | LCP / FID / CLS / INP |
| JS 错误 | 开 | window.onerror + unhandledrejection |
| 资源错误 | 开 | error event capture |
| 停留时长 | 开 | visibilitychange |
| 全局点击 | 关 | 启用后所有点击自动上报 |

### 上报通道（优先级）

```
1. navigator.sendBeacon (首选，页面关闭也能发)
2. fetch(keepalive: true) (备选)
3. XMLHttpRequest (最后兜底)
```

### 离线缓存

```
失败 / 离线
   ↓
IndexedDB 存储（封装 idb-keyval）
   ↓
最多缓存 1000 条，超出 FIFO 丢弃
   ↓
网络恢复 / 下次打开 → 自动重发
```

### 安全要求

- AppSecret 绝不出现在前端
- Token 模式：业务后端签发 ≤ 15min 短期 Token
- 敏感字段（如手机号）自动脱敏
- 同源策略 + CORS 严格配置

## 三、版本管理

### SemVer

- 主版本变更：API 破坏性
- 次版本变更：新增能力
- 补丁版本：bug 修复

### 兼容期

- 至少 N-2 版本兼容
- 废弃 API 提前 2 个次版本警告
- 重大变更走灰度发布

### Release Note

每个版本必须有：
- 新增功能
- 废弃 API
- Bug 修复
- 升级指南

## 四、文档与样板工程

| 资源 | 说明 |
|---|---|
| 接入文档 | 30 分钟可接入 |
| Java 样板工程 | `analytics-demo-springboot` |
| Vue3 样板工程 | `analytics-demo-vue3` |
| 接入大使 | 企微群驻点答疑 |
| FAQ | 常见问题集 |
