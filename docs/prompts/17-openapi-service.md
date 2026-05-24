# 提示词 17 · OpenAPI 服务

> 前置：`00-master-prompt.md`。  
> 参考：`docs/12-api-spec/`。

---

## 提示词正文

```
你是 API 平台工程师。请开发 analytics-openapi 服务。

【目标】
对外暴露统一开放 API：业务系统消费指标 / 标签 / 分群 / 触达 / 决策能力。

【技术】

- Spring Boot 3.2 + WebFlux 或 MVC（看场景）
- Spring Cloud Gateway（已在 gateway 服务）
- SpringDoc OpenAPI 2.x（自动生成规范）
- openapi-generator（自动生成 SDK Client）
- Sentinel（限流）
- Redis（幂等 + Nonce）

【接口路径规范】

```
https://api.analytics.yourcompany.com/openapi/v{version}/{resource}/{action}
```

详见 docs/12-api-spec/01-openapi-spec.md 完整清单。

【鉴权机制（详细实现）】

```java
@Component
public class AppKeySignFilter implements GlobalFilter {
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest req = exchange.getRequest();
        
        String appKey = req.getHeaders().getFirst("X-App-Key");
        String timestamp = req.getHeaders().getFirst("X-Timestamp");
        String nonce = req.getHeaders().getFirst("X-Nonce");
        String signature = req.getHeaders().getFirst("X-Signature");
        
        // 1. 必需头校验
        if (any null) return reject(401, "缺少鉴权头");
        
        // 2. 时间漂移校验（5 分钟）
        long ts = Long.parseLong(timestamp);
        if (Math.abs(System.currentTimeMillis() / 1000 - ts) > 300) {
            return reject(401, "请求过期");
        }
        
        // 3. Nonce 防重放（Redis 5min）
        String nonceKey = "analytics:openapi:nonce:" + nonce;
        if (!redisTemplate.opsForValue().setIfAbsent(nonceKey, "1", Duration.ofMinutes(5))) {
            return reject(401, "Nonce 重复");
        }
        
        // 4. 查 AppSecret
        String appSecret = appCredentialService.getSecret(appKey);
        if (appSecret == null) return reject(401, "AppKey 无效");
        
        // 5. 计算签名
        return readBody(req).flatMap(body -> {
            String bodyMd5 = body.isEmpty() ? "" : md5(body);
            String signString = req.getMethod() + "\n" + req.getPath() + "\n" + timestamp + "\n" + nonce + "\n" + bodyMd5;
            String expectedSign = base64(hmacSha256(signString, appSecret));
            
            if (!expectedSign.equals(signature)) {
                return reject(401, "签名错误");
            }
            
            // 注入 AppKey 到下游
            exchange.getRequest().mutate().header("X-Internal-App-Key", appKey);
            return chain.filter(exchange);
        });
    }
}
```

【限流（Sentinel）】

```
按 AppKey + 接口 双维度
按 IP 兜底
配额：
  free tier: 100 QPS / 分钟
  paid tier: 1000+ QPS / 分钟
超限返回 429 + Retry-After
```

【幂等控制】

```java
@Component
public class IdempotencyFilter {
    public Mono<ResponseEntity<?>> handle(ServerWebExchange exchange, ChainResult result) {
        String idempotencyKey = exchange.getRequest().getHeaders().getFirst("X-Idempotency-Key");
        if (isWriteOperation(exchange) && idempotencyKey == null) {
            return reject(400, "写操作必须 X-Idempotency-Key");
        }
        
        String cacheKey = "analytics:openapi:idempotency:" + idempotencyKey;
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return Mono.just((ResponseEntity<?>) cached);  // 返回缓存结果
        }
        
        return executeAndCache(exchange, cacheKey, Duration.ofHours(24));
    }
}
```

【响应规范】

```java
@RestControllerAdvice
public class ResponseAdvice {
    @ExceptionHandler(BusinessException.class)
    public Result<?> handle(BusinessException ex) {
        return Result.error(ex.getCode(), ex.getMessage())
            .withTraceId(MDC.get("traceId"))
            .withRequestId(MDC.get("requestId"));
    }
}
```

【接口实现】

每个 API 必须：
1. @Validated DTO 参数校验
2. @PreAuthorize 或行级权限拦截
3. SpringDoc 注解（@Operation / @Parameter / @ApiResponse）
4. 调用下游服务（指标 / 分群 / 触达 / 决策）走 Feign
5. 异常处理统一

【SDK 自动生成】

```bash
# 在 CI 中自动生成
openapi-generator generate -i openapi.yaml -g java -o sdks/java
openapi-generator generate -i openapi.yaml -g typescript-axios -o sdks/typescript
openapi-generator generate -i openapi.yaml -g python -o sdks/python

# 发布到公司私服
mvn deploy / npm publish / twine upload
```

【沙箱环境】

- 独立部署一套（sandbox.analytics.yourcompany.com）
- 数据完全隔离（独立租户 ID 段）
- 与 prod 同 API 规范
- 配额放宽

【开发者门户】

独立前端项目 / 子模块：
- 应用管理（创建 / 查看 AppKey）
- AppSecret 重置
- 调用统计
- 文档（Swagger UI 嵌入）
- 在线调试
- 错误码字典
- SDK 下载

【版本管理】

- URL 路径版本号
- 老版本至少 6 个月
- 废弃接口返回 Deprecation + Sunset Header
- 自动通知接入方迁移

【监控】

- 按 AppKey 维度统计调用 QPS / 成功率 / P95
- 按接口维度
- 异常告警
- 限流告警

【性能】

- 简单查询 P95 ≤ 500ms
- 复杂查询 P95 ≤ 5s
- 鉴权耗时 < 20ms（缓存命中）

【验证】

1. HMAC 签名正确性
2. 防重放（Nonce 5 分钟）
3. 时间漂移防护
4. 限流生效
5. 幂等生效
6. SDK 自动生成
7. 沙箱与 prod 隔离

【交付物】

- 完整服务代码
- OpenAPI 3.0 yaml
- 自动生成的 SDK（Java / TS / Python）
- 开发者门户
- 接入文档
- 性能压测报告

【禁止】

- 跳过签名校验
- AppSecret 明文存储
- 错误响应透传内部堆栈
- 写接口无幂等
- 限流配额硬编码

【安全】

- AppSecret KMS 加密
- 所有调用审计
- 异常 AppKey 自动冻结
- 防 DDoS

现在请生成。
```
