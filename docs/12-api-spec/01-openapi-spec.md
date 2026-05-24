# OpenAPI 规范

## 一、通用规范

### URL 结构

```
https://api.analytics.yourcompany.com/openapi/v{version}/{resource}/{action}

例：
GET  /openapi/v1/metric/query
POST /openapi/v1/segment/preview
GET  /openapi/v1/user/profile/{userId}
```

### 版本管理

- URL 路径版本号：`/v1/`、`/v2/`
- 老版本至少保留 6 个月
- 废弃接口返回 `Deprecation` + `Sunset` Header

### 响应格式

```json
{
  "code": 0,
  "message": "ok",
  "data": { ... },
  "traceId": "abc123",
  "requestId": "req-456"
}
```

### 错误码

```
6 位数字：前 2 位服务码 + 后 4 位业务码

服务码：
  10xxxx - gateway
  11xxxx - auth
  12xxxx - metric
  13xxxx - segment
  14xxxx - tag
  15xxxx - reach
  16xxxx - experiment
  17xxxx - report
  18xxxx - assistant
  19xxxx - decision

业务码（示例）：
  120001 - 指标不存在
  120002 - 指标查询超时
  120003 - 指标查询参数错误
  120401 - 无指标权限
  120429 - 触发限流
```

### 分页

```
游标分页（cursor-based），避免深分页性能问题

请求：
  ?cursor=xxx&limit=100  (max 200)

响应：
  {
    "items": [...],
    "nextCursor": "yyy",
    "hasMore": true
  }
```

### 时间字段

- 统一 ISO 8601 + 时区：`2026-05-23T10:00:00+08:00`
- 时间戳：毫秒（13 位）

## 二、核心 API 清单（V1）

### 鉴权

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/auth/exchange` | POST | Token 交换（SSO 打通） |
| `/openapi/v1/auth/refresh` | POST | 刷新 Token |

### 指标查询

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/metric/query` | POST | 指标查询（DSL） |
| `/openapi/v1/metric/{code}` | GET | 指标元数据 |
| `/openapi/v1/metric/list` | GET | 指标列表 |

### 用户/企业画像

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/user/profile/{userId}` | GET | 用户画像 |
| `/openapi/v1/org/profile/{orgId}` | GET | 企业画像 |
| `/openapi/v1/user/{userId}/timeline` | GET | 用户行为时间线 |
| `/openapi/v1/user/{userId}/tags` | GET | 用户标签 |

### 分群

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/segment/list` | GET | 分群列表 |
| `/openapi/v1/segment/{id}/users` | GET | 分群用户列表 |
| `/openapi/v1/segment/preview` | POST | 分群人数预估 |
| `/openapi/v1/segment` | POST | 创建分群 |
| `/openapi/v1/segment/{id}` | DELETE | 删除分群 |

### 事件

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/event/track` | POST | 单事件上报（备用，主推 SDK） |
| `/openapi/v1/event/batch` | POST | 批量事件上报 |

### Journey / 触达

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/journey/trigger` | POST | 触发 Journey |
| `/openapi/v1/reach/send` | POST | 即时触达 |

### AB 实验

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/experiment/{expId}/variant` | GET | 用户实验分组 |
| `/openapi/v1/experiment/{expId}/result` | GET | 实验结果 |

### 智能洞察

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/insight/anomaly` | GET | 最近异动列表 |
| `/openapi/v1/insight/anomaly/{id}` | GET | 异动详情 + 归因 |

### 决策追溯

| 路径 | 方法 | 说明 |
|---|---|---|
| `/openapi/v1/decision` | POST | 登记决策 |
| `/openapi/v1/decision/{id}` | GET | 决策详情 |
| `/openapi/v1/decision/{id}/review` | POST | 提交复盘 |
| `/openapi/v1/decision/search` | POST | 检索相似决策 |

## 三、关键接口详细规约

### POST /openapi/v1/metric/query

```yaml
请求体:
  metric: string         # 指标 code，必填
  dimensions: string[]   # 维度
  filters:
    - field: string
      op: '=' | '!=' | '>' | '<' | 'in' | 'between'
      value: any
  dateRange:
    start: ISO8601
    end: ISO8601
  granularity: 'hour' | 'day' | 'week' | 'month'
  compare: 'mom' | 'yoy' | 'custom'
  limit: number  # default 100
  cursor: string

响应:
  data:
    metric: string
    rows:
      - dimensions: { ... }
        value: number
        compareValue?: number
        changeRate?: number
    total: number
    schema: { ... }
```

### POST /openapi/v1/segment/preview

```yaml
请求体:
  rules:
    operator: 'and' | 'or'
    conditions:
      - type: 'event' | 'attribute' | 'tag'
        field: string
        op: string
        value: any

响应:
  data:
    estimatedCount: number
    sampleUsers: [...]  # 10 个示例
    breakdown: { ... }  # 按维度拆分
```

### GET /openapi/v1/user/profile/{userId}

```yaml
路径参数:
  userId: string

查询参数:
  include?: 'basic,tags,timeline,scores,actions'

响应:
  data:
    basic: { name, email, role, org }
    tags: [...]
    healthScore: number
    churnRisk: number
    ltv: number
    timeline: [...]  # 最近 30 天
    suggestedActions: [...]
```

## 四、SDK 自动生成

使用 `openapi-generator` 自动生成多语言 SDK：

```bash
# Java Client
openapi-generator generate -i analytics-openapi.yaml -g java -o ./sdk-java

# TypeScript Client
openapi-generator generate -i analytics-openapi.yaml -g typescript-axios -o ./sdk-ts

# Python Client
openapi-generator generate -i analytics-openapi.yaml -g python -o ./sdk-python
```

发布到公司私服仓库。

## 五、沙箱与生产

| 环境 | 域名 |
|---|---|
| 沙箱 | sandbox.analytics.yourcompany.com |
| 生产 | api.analytics.yourcompany.com |

沙箱数据完全隔离，可放心测试。

## 六、开发者门户

提供独立的 Developer Portal：
- 应用申请
- 密钥管理（AppKey / AppSecret 自助生成与轮换）
- 调用统计
- 文档（Swagger UI + 示例）
- 在线调试
- 错误码字典
- SDK 下载

## 七、性能要求

| 接口类别 | P95 响应时间 |
|---|---|
| 简单查询（指标元数据 / 用户基础信息） | ≤ 200ms |
| 指标查询 | ≤ 500ms |
| 分群预估 | ≤ 2s |
| 即席查询 | ≤ 5s |
| 写操作（决策登记 / 创建分群） | ≤ 1s |

## 八、SLA

- 可用性 ≥ 99.95%
- 故障响应 ≤ 30min
- 故障通告：实时（status.analytics.yourcompany.com）
