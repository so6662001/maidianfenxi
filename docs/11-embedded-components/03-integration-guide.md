# 业务方接入指南

## 一、接入流程

```
1. 业务方申请接入
   ↓
2. 中台分配 AppKey + AppSecret + 接入文档
   ↓
3. 业务方选择接入方式（数据上报 SDK / OpenAPI / 嵌入组件，可组合）
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

## 二、3 种接入方式选择

| 方式 | 适用 | 接入工时 | 维护成本 |
|---|---|---|---|
| **数据上报 SDK** | 需要把业务数据/行为送入中台 | 1-2 天 | 低 |
| **OpenAPI** | 业务系统需要查指标/拉标签/触发分群 | 1-2 天 | 低 |
| **嵌入组件** | 业务系统需要展示画像/分析能力 | 0.5-1 天 | 极低 |

可组合使用，例如 CRM 系统：
- SDK：上报客户行为
- OpenAPI：查询客户画像 / 流失风险
- 嵌入：`<UserProfile360 />` 直接展示

## 三、Java SDK 接入示例

### Spring Boot 项目

```xml
<!-- pom.xml -->
<dependency>
  <groupId>com.yourcompany.analytics</groupId>
  <artifactId>analytics-spring-boot-starter</artifactId>
  <version>1.0.0</version>
</dependency>
```

```yaml
# application.yml
analytics:
  enabled: true
  endpoint: https://collector.yourcompany.com
  app-key: ${ANALYTICS_APP_KEY}
  app-secret: ${ANALYTICS_APP_SECRET}
```

```java
@SpringBootApplication
@EnableAnalytics
public class CrmApplication { ... }

@Service
public class CustomerService {
    @Autowired AnalyticsClient analytics;
    
    public Customer createCustomer(CustomerRequest req) {
        Customer c = ...;
        analytics.track(currentUser.getId(), "crm_customer_create", Map.of(
            "customerId", c.getId(),
            "industry", c.getIndustry()
        ));
        return c;
    }
}
```

### 非 Spring Java 项目

```java
AnalyticsClient client = AnalyticsClientBuilder.newBuilder()
    .endpoint("https://collector.yourcompany.com")
    .appKey(System.getenv("ANALYTICS_APP_KEY"))
    .appSecret(System.getenv("ANALYTICS_APP_SECRET"))
    .batchSize(200)
    .flushIntervalMs(2000)
    .build();

// 业务代码
client.track("user-123", "page_view", Map.of("page", "/home"));

// 关闭
Runtime.getRuntime().addShutdownHook(new Thread(client::close));
```

## 四、Vue3 SDK 接入示例

```bash
pnpm add @yourcompany/analytics-sdk
```

```ts
// main.ts
import { createApp } from 'vue';
import App from './App.vue';
import { AnalyticsPlugin } from '@yourcompany/analytics-sdk';

const app = createApp(App);

app.use(AnalyticsPlugin, {
  endpoint: 'https://collector.yourcompany.com',
  appKey: import.meta.env.VITE_ANALYTICS_APP_KEY,
  getToken: async () => {
    const res = await fetch('/api/get-analytics-token', {
      credentials: 'include'
    });
    return await res.text();
  },
  autoTrack: { pv: true, error: true, performance: true },
  debug: import.meta.env.DEV
});

app.mount('#app');
```

```vue
<!-- 任意组件 -->
<template>
  <button v-track="'invoice_export_click'">导出发票</button>
</template>

<script setup>
import { useAnalytics } from '@yourcompany/analytics-sdk';
const { track } = useAnalytics();

function onSomeAction() {
  track('custom_event', { foo: 'bar' });
}
</script>
```

## 五、OpenAPI 接入示例

```ts
// 业务后端代理（前端不存 Secret）
import { createHmac } from 'crypto';

async function queryUserProfile(userId: string) {
  const timestamp = Math.floor(Date.now() / 1000);
  const nonce = crypto.randomUUID();
  const path = `/openapi/v1/user/profile/${userId}`;
  const signString = `GET\n${path}\n${timestamp}\n${nonce}\n`;
  const signature = createHmac('sha256', APP_SECRET)
    .update(signString)
    .digest('base64');
  
  const res = await fetch(`https://api.analytics.yourcompany.com${path}`, {
    headers: {
      'X-App-Key': APP_KEY,
      'X-Timestamp': String(timestamp),
      'X-Nonce': nonce,
      'X-Signature': signature
    }
  });
  
  return await res.json();
}
```

## 六、嵌入组件接入示例

```bash
pnpm add @yourcompany/analytics-vue-components
```

```ts
// main.ts 全局配置一次
import { setupAnalyticsComponents } from '@yourcompany/analytics-vue-components';

setupAnalyticsComponents({
  endpoint: 'https://api.analytics.yourcompany.com',
  getToken: async () => {
    const res = await fetch('/api/get-analytics-token');
    return await res.text();
  }
});
```

```vue
<!-- CRM 客户详情页直接嵌入 -->
<template>
  <div class="customer-detail">
    <BasicInfo :customer="customer" />
    
    <!-- 中台用户 360° 组件 -->
    <UserProfile360 
      :user-id="customer.userId"
      :org-context="customer.orgId"
      layout="full"
      @action="handleAction"
    />
    
    <!-- 流失风险卡 -->
    <ChurnRiskCard :user-id="customer.userId" />
  </div>
</template>

<script setup>
import { UserProfile360, ChurnRiskCard } from '@yourcompany/analytics-vue-components';

function handleAction(type: string, payload: any) {
  if (type === 'add-to-segment') {
    // 处理业务逻辑
  }
}
</script>
```

## 七、SSO 打通

业务系统需提供以下接口：

```
GET /api/get-analytics-token
  鉴权：业务系统 SSO Session
  Response: { token: "短期 JWT ≤ 15min" }
```

中台前端调用时自动用此 Token 换取中台 JWT，实现"用户无感登录中台"。

## 八、沙箱与生产

| 环境 | 域名 | 用途 |
|---|---|---|
| 沙箱 | sandbox.analytics.yourcompany.com | 接入联调，数据隔离 |
| 生产 | api.analytics.yourcompany.com | 正式调用 |

**切到生产 checklist**：
- [ ] 沙箱已跑通核心流程
- [ ] 错误率 < 1%
- [ ] 埋点覆盖率 ≥ 90%
- [ ] AppKey/AppSecret 已切换
- [ ] 灰度发布（先 10% → 50% → 100%）
- [ ] 监控告警已配置

## 九、SLA 与支持

| 项 | 标准 |
|---|---|
| 接入响应 | T+1 工作日 |
| 平均接入工时 | 目标 ≤ 3 个工作日 |
| 沙箱可用性 | 7×24，与 prod 数据隔离 |
| 接入大使支持 | 企微群驻点答疑 |
| 文档完整度 | 100% 接口含示例 |
| SDK 兼容期 | 至少 N-2 版本 |

## 十、常见问题

### Q1: SDK 上报失败会丢数据吗？
A: 失败后会本地落盘（Chronicle Queue / IndexedDB），下次启动自动重试。

### Q2: 嵌入组件能跨域吗？
A: 可以，中台启用 CORS + 短期 Token 鉴权。

### Q3: 一定要用 OneID 吗？
A: 必须。如果业务方有自己的用户体系，中台会自动建立映射。

### Q4: 业务数据怎么补传？
A: 通过 OpenAPI 的 `/event/track` 接口批量补传，或走业务库 CDC。

### Q5: 我的业务系统是 Vue2 / React，怎么接入嵌入组件？
A: 使用 Web Components 形式或 wujie 微前端嵌入。

### Q6: 中台升级会影响我吗？
A: 严格 SemVer + N-2 兼容，重大变更提前 2 个次版本警告。
