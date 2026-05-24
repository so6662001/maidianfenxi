# 鉴权与安全

## 一、三种鉴权方式

| 场景 | 方式 |
|---|---|
| 内部用户访问 Web | SSO + JWT |
| 业务系统调用 OpenAPI | AppKey + HMAC 签名 |
| 嵌入式组件 / 微前端 | 短期 Token（业务后端签发） |

## 二、SSO + JWT

```
用户 → 公司 IAM 登录
   ↓
IAM 颁发 SAML / OIDC Assertion
   ↓
中台 analytics-auth 验证 + 颁发 JWT
   JWT Payload: { userId, tenantId, roles, exp }
   ↓
后续请求带 Authorization: Bearer {JWT}
   ↓
JWT 1h 过期 + Refresh Token 7d
```

## 三、AppKey + HMAC 签名（OpenAPI）

### 申请流程

1. 业务方在开发者门户申请应用
2. 系统生成 AppKey + AppSecret（一次显示，需妥善保存）
3. 业务方在调用时计算签名

### 签名算法

```
sign_string = METHOD + "\n" + 
              PATH + "\n" + 
              TIMESTAMP + "\n" + 
              NONCE + "\n" + 
              MD5(BODY)  # GET 请求为空

signature = Base64( HMAC-SHA256(sign_string, APP_SECRET) )

Headers:
  X-App-Key: <APP_KEY>
  X-Timestamp: <Unix 秒>
  X-Nonce: <UUID>
  X-Signature: <signature>
  X-Idempotency-Key: <UUID>  # 写操作必填
```

### 防重放

- Timestamp 与服务端时间差 ≤ 5 分钟
- Nonce 5 分钟内不可重复（Redis 缓存校验）

### 示例代码

```java
String timestamp = String.valueOf(System.currentTimeMillis() / 1000);
String nonce = UUID.randomUUID().toString();
String bodyMd5 = body == null ? "" : md5(body);
String signString = method + "\n" + path + "\n" + timestamp + "\n" + nonce + "\n" + bodyMd5;
String signature = base64(hmacSha256(signString, appSecret));

Request req = new Request.Builder()
    .url(endpoint + path)
    .method(method, body == null ? null : RequestBody.create(body, JSON))
    .header("X-App-Key", appKey)
    .header("X-Timestamp", timestamp)
    .header("X-Nonce", nonce)
    .header("X-Signature", signature)
    .header("X-Idempotency-Key", UUID.randomUUID().toString())
    .build();
```

```ts
async function callOpenApi(method: string, path: string, body?: any) {
  const timestamp = Math.floor(Date.now() / 1000).toString();
  const nonce = crypto.randomUUID();
  const bodyMd5 = body ? md5(JSON.stringify(body)) : '';
  const signString = `${method}\n${path}\n${timestamp}\n${nonce}\n${bodyMd5}`;
  const signature = base64(hmacSha256(signString, APP_SECRET));
  
  return fetch(endpoint + path, {
    method,
    headers: {
      'X-App-Key': APP_KEY,
      'X-Timestamp': timestamp,
      'X-Nonce': nonce,
      'X-Signature': signature,
      'X-Idempotency-Key': crypto.randomUUID(),
      'Content-Type': 'application/json'
    },
    body: body ? JSON.stringify(body) : undefined
  });
}
```

## 四、嵌入式短期 Token

```
业务系统主框架已登录
   ↓
业务后端调用中台 /openapi/v1/auth/sign-embed-token
   传入：业务系统用户身份 + 嵌入场景
   返回：短期 Token（默认 15min）
   ↓
业务后端把 Token 返回给前端
   ↓
前端嵌入组件用 Token 调用中台 API
   ↓
中台验证 Token，返回数据
   ↓
Token 过期前自动刷新
```

### 业务后端接入接口

```yaml
GET /api/get-analytics-token
鉴权: 业务系统 SSO Session
逻辑:
  - 验证用户身份
  - 调用中台 /openapi/v1/auth/sign-embed-token
  - 返回短期 Token
Response: { token: "xxx", expiresAt: 1234567890 }
```

## 五、权限控制

### RBAC 三级

```
用户 → 角色 → 权限
       ↓
   资源 (看板 / API / 分群 / 决策)
       ↓
   操作 (读 / 写 / 管理)
```

### 行级权限

所有查询自动注入：

```sql
WHERE tenant_id = :currentTenantId
  AND (:isAdmin OR org_id IN (:userOrgIds))
  AND (:isAdmin OR product_line IN (:userProductLines))
```

### 字段级脱敏

```
敏感字段（手机号 / 身份证 / 订单金额）按角色脱敏：
  超级管理员: 138****8888 / ***
  普通管理员: 138****8888 / 显示 ¥xxx
  业务员: 完全隐藏
```

## 六、限流

### Sentinel 多维度

```
按 IP: 100 QPS / min
按 AppKey: 1000 QPS / min
按 接口: 各接口独立配额
按 用户: 100 QPS / min（防爬虫）
```

### 限流响应

```http
HTTP 429 Too Many Requests
Retry-After: 60
{
  "code": 120429,
  "message": "请求过于频繁，请稍后重试",
  "traceId": "..."
}
```

## 七、审计日志

### 必须记录

- 所有数据访问（who / when / what / IP / UA）
- 所有数据导出（含水印）
- 所有决策操作（创建 / 修改 / 删除）
- 所有权限变更
- 所有 AppKey/AppSecret 生成/轮换

### 存储

- 写入 ES + ClickHouse
- 保留 ≥ 180 天
- 不可篡改

### 查询

```
GET /openapi/v1/audit/log
  权限：仅审计员 / 安全员
  Query: userId, resource, dateRange, action
```

## 八、敏感数据保护

| 数据 | 保护 |
|---|---|
| AppSecret | 加密存储（KMS / Vault），仅创建时显示 |
| 手机号 | 字段级加密 + 脱敏展示 |
| 身份证 | 同上 |
| 订单金额 | 按角色脱敏 |
| Token | 仅 Redis 缓存，TTL ≤ 1h |
| 密码 | bcrypt（成本因子 ≥ 10） |

## 九、合规

### 个人信息保护法（PIPL）

- 用户授权链路完整（注册时告知 + 同意）
- 提供"数据导出" + "数据删除"自助接口
- 跨境传输评估（如有海外用户）

### 数据安全法

- 数据分级（公开 / 内部 / 敏感 / 极敏感）
- 重要数据出境评估
- 数据安全事件应急响应预案

### 隐私评估（PIA）

每个新功能上线前必须做 PIA 评估，输出报告：
- 收集什么数据
- 使用目的
- 存储期限
- 访问权限
- 风险评估

## 十、安全测试

| 测试 | 频率 |
|---|---|
| 依赖漏洞扫描（Snyk / OWASP DC） | 每次 PR |
| SAST 静态扫描（SonarQube / Checkmarx） | 每次 PR |
| DAST 动态扫描 | 每次发版 |
| 渗透测试 | 季度 |
| 红蓝对抗 | 年度 |
