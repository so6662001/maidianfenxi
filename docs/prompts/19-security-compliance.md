# 提示词 19 · 安全合规审计

> 前置：`00-master-prompt.md`。  
> 参考：`docs/01-requirements/04-non-functional-requirements.md` + `docs/12-api-spec/02-authentication.md`。

---

## 提示词正文

```
你是安全工程师。请为 analytics-platform 实施完整的安全合规体系。

【目标】
实现 PIPL / 数据安全法 / GDPR 合规 + 防 OWASP Top 10 + 完整审计追溯。

【涵盖范围】

1. 鉴权与授权
2. 数据保护（传输 + 存储）
3. 输入校验 + 防注入
4. 审计日志
5. 合规（PIPL / GDPR）
6. 安全测试
7. 应急响应

【鉴权与授权】

### JWT 配置

```java
@Configuration
public class JwtConfig {
    @Value("${jwt.secret}") String secret;   // 从 Vault 读取
    @Value("${jwt.access-token-ttl}") long accessTtl = 3600;
    @Value("${jwt.refresh-token-ttl}") long refreshTtl = 604800;
    
    public String generateAccessToken(User user) {
        return JWT.create()
            .withSubject(user.getId())
            .withClaim("tenantId", user.getTenantId())
            .withClaim("roles", user.getRoles())
            .withIssuedAt(now())
            .withExpiresAt(now().plusSeconds(accessTtl))
            .sign(Algorithm.HMAC256(secret));
    }
    
    public void revokeToken(String token) {
        redis.opsForValue().set("blacklist:" + tokenHash, "1", Duration.ofSeconds(accessTtl));
    }
}
```

### RBAC + 行级权限

```java
@Aspect
@Component
public class DataPermissionAspect {
    @Around("@annotation(EnableDataPermission)")
    public Object inject(ProceedingJoinPoint pjp) {
        UserContext ctx = SecurityContext.current();
        DataPermissionContext.set(
            ctx.getTenantId(),
            ctx.getOrgIds(),
            ctx.getProductLines(),
            ctx.isAdmin()
        );
        try {
            return pjp.proceed();
        } finally {
            DataPermissionContext.clear();
        }
    }
}

// MyBatis 拦截器自动注入 WHERE 条件
@Intercepts(...)
public class DataPermissionInterceptor implements Interceptor {
    public Object intercept(Invocation invocation) {
        // 改写 SQL 注入 tenant_id + org_id + product_line 过滤
    }
}
```

### 字段级脱敏

```java
@Target(ElementType.FIELD)
public @interface Sensitive {
    SensitiveType type();
    String[] visibleRoles() default {};
}

public class User {
    @Sensitive(type = SensitiveType.PHONE, visibleRoles = {"ADMIN", "SECURITY"})
    private String phone;
    
    @Sensitive(type = SensitiveType.ID_CARD)
    private String idCard;
}

// 序列化时自动脱敏
public class SensitiveSerializer extends JsonSerializer<String> {
    public void serialize(String value, JsonGenerator gen, SerializerProvider sp) {
        UserContext ctx = SecurityContext.current();
        if (canSeeFullValue(ctx, field)) {
            gen.writeString(value);
        } else {
            gen.writeString(maskValue(value, field.type()));
        }
    }
}
```

### 加密存储

```java
@Convert(converter = EncryptedStringConverter.class)
@Column(name = "phone")
private String phone;

public class EncryptedStringConverter implements AttributeConverter<String, String> {
    private final KmsClient kms;  // AWS KMS / 阿里云 KMS / Vault Transit
    
    public String convertToDatabaseColumn(String value) {
        return kms.encrypt(value);
    }
    
    public String convertToEntityAttribute(String encrypted) {
        return kms.decrypt(encrypted);
    }
}
```

【输入校验 + 防注入】

```java
@PostMapping("/metric/query")
public Result<MetricResult> query(@Valid @RequestBody MetricQueryRequest req) {
    // @Valid 触发 Jakarta Validation
    return Result.success(metricService.query(req));
}

public class MetricQueryRequest {
    @NotBlank(message = "metric 不能为空")
    @Pattern(regexp = "^M-[A-Z]+-\\d{3}$", message = "metric code 格式错误")
    private String metric;
    
    @Size(max = 10, message = "维度最多 10 个")
    private List<@Pattern(regexp = "^[a-z_]+$") String> dimensions;
    
    @NotNull @Valid
    private DateRange dateRange;
}
```

**所有 SQL 必须参数化**：

```java
// ❌ 禁止
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// ✅ MyBatis 自动参数化
@Select("SELECT * FROM users WHERE name = #{name}")
User findByName(@Param("name") String name);

// ✅ Calcite / ClickHouse JDBC 参数化
PreparedStatement ps = conn.prepareStatement("SELECT ... WHERE ? = ?");
```

【XSS 防护】

```java
// 全局响应 Header
@Configuration
public class SecurityHeadersConfig {
    @Bean
    public WebMvcConfigurer corsConfig() {
        return new WebMvcConfigurer() {
            public void addCorsMappings(CorsRegistry r) {
                r.addMapping("/**")
                    .allowedOrigins(...)
                    .allowedMethods("GET", "POST", "PUT", "DELETE")
                    .maxAge(3600);
            }
        };
    }
    
    @Bean
    public FilterRegistrationBean<SecurityHeadersFilter> headersFilter() {
        // 添加：
        // Content-Security-Policy
        // X-Content-Type-Options: nosniff
        // X-Frame-Options: DENY
        // Strict-Transport-Security: max-age=31536000
        // Referrer-Policy: strict-origin-when-cross-origin
    }
}

// 前端 XSS 防护：
// - Vue 默认 {{ }} 转义
// - v-html 严格审核
// - DOMPurify 清理用户输入
```

【CSRF 防护】

```
JWT 请求自动免疫 CSRF（Authorization Header 不会被自动带）
但 Cookie 模式需要 CSRF Token
```

【审计日志】

```java
@Aspect
@Component
public class AuditAspect {
    @Around("@annotation(Auditable)")
    public Object audit(ProceedingJoinPoint pjp) {
        UserContext ctx = SecurityContext.current();
        AuditEvent event = new AuditEvent();
        event.setUserId(ctx.getUserId());
        event.setAction(method.getName());
        event.setResource(...);
        event.setIp(request.getRemoteAddr());
        event.setUserAgent(...);
        event.setTraceId(MDC.get("traceId"));
        event.setStartTime(now());
        
        try {
            Object result = pjp.proceed();
            event.setStatus("SUCCESS");
            return result;
        } catch (Exception ex) {
            event.setStatus("FAILED");
            event.setError(ex.getMessage());
            throw ex;
        } finally {
            event.setEndTime(now());
            auditEventPublisher.publish(event);  // 异步写 ES
        }
    }
}

// 关键接口标注
@Auditable(category = "DATA_ACCESS", level = "HIGH")
@GetMapping("/user/profile/{userId}")
public Result<UserProfile> getProfile(@PathVariable String userId) { ... }
```

### 审计存储

- 写 Elasticsearch（保留 180 天）+ ClickHouse 长期存档
- 不可篡改（write-only）
- 定期备份

【限流防 DDoS】

```yaml
sentinel:
  rules:
    - resource: /openapi/**
      grade: QPS
      count: 1000
      strategy: 0  # 直接
      controlBehavior: 0  # 快速失败
    - resource: /api/assistant/chat
      grade: QPS
      count: 5      # 单用户
      strategy: 1   # 关联（按用户）
```

【合规：PIPL 必备功能】

```java
@RestController
@RequestMapping("/api/privacy")
public class PrivacyController {
    // 数据导出
    @PostMapping("/export")
    public Result<String> exportMyData(@AuthenticationPrincipal User user) {
        return Result.success(privacyService.exportUserData(user.getId()));
    }
    
    // 数据删除（GDPR Right to Delete）
    @PostMapping("/delete")
    public Result<Void> deleteMyData(@AuthenticationPrincipal User user) {
        privacyService.deleteUserData(user.getId());
        return Result.success();
    }
    
    // 数据访问记录
    @GetMapping("/access-log")
    public Result<List<AccessLog>> myAccessLog(@AuthenticationPrincipal User user) { ... }
    
    // 撤销同意
    @PostMapping("/consent/revoke")
    public Result<Void> revokeConsent(...) { ... }
}
```

【数据生命周期】

| 数据 | 热存 | 冷存 | 归档/删除 |
|---|---|---|---|
| 行为明细 | 24m | 36m | 删除 |
| 聚合数据 | 36m | 永久 | — |
| 元数据 | 永久 | — | — |
| 决策记录 | 永久 | — | — |
| 操作日志 | 6m | 24m | 删除 |
| LLM 对话 | 3m | 12m | 删除（PII 脱敏） |

定时任务自动归档/删除。

【密钥管理】

- HashiCorp Vault 或云 KMS
- 应用通过 Vault Agent 获取
- 自动轮换：API Key 90 天，AppSecret 用户可主动重置
- 密钥不出现在任何日志 / 代码 / 配置

【安全测试】

```yaml
# CI 集成
依赖漏洞扫描:
  - OWASP Dependency Check（每次 PR）
  - Snyk

SAST 静态扫描:
  - SonarQube
  - Checkmarx
  
DAST 动态扫描:
  - OWASP ZAP（每次发版前）
  
镜像扫描:
  - Trivy（push 镜像前）

渗透测试:
  - 季度一次（外部团队）
  
红蓝对抗:
  - 年度一次
```

【应急响应预案】

```
1. 数据泄露
   ↓ 第一步：阻断（关闭涉事 API / 冻结涉事账号）
   ↓ 第二步：评估影响范围
   ↓ 第三步：通知相关方
   ↓ 第四步：监管报告（72h 内，按 GDPR / 数据安全法）
   ↓ 第五步：根因分析 + 改进

2. 服务被入侵
   ↓ 隔离受影响 Pod
   ↓ 保留现场（不立即重启）
   ↓ 调取审计日志
   ↓ 系统恢复
   ↓ 根因分析 + 加固

3. DDoS 攻击
   ↓ WAF / CDN 防护
   ↓ 限流策略升级
   ↓ 黑名单 IP
```

【PIA 评估流程】

每个新功能上线前必须输出 PIA 报告：
- 收集什么数据
- 使用目的
- 存储期限
- 访问权限
- 风险评估
- 缓解措施

【验收】

1. 所有接口必须 JWT 或 HMAC 鉴权
2. 所有 SQL 参数化（静态扫描 0 命中）
3. 所有敏感字段加密 + 脱敏
4. 所有重要操作有审计日志
5. PIPL / GDPR 合规接口齐全
6. 密钥不在代码/配置/日志中
7. OWASP ZAP 扫描无高危漏洞
8. Trivy 镜像扫描无 Critical
9. 渗透测试无重大问题

【交付物】

- 完整鉴权代码（JwtConfig / HmacFilter / RBAC）
- 字段脱敏注解 + 序列化器
- 加密存储 Converter
- 审计 AOP + ES 写入
- 限流 Sentinel 规则
- 合规接口（导出 / 删除 / 同意管理）
- PIA 评估模板
- 应急响应手册
- 安全测试脚本（OWASP ZAP）
- 月度安全报告模板

【禁止】

- 明文密码 / 密钥
- 字符串拼 SQL
- 跳过权限校验
- 错误响应透传堆栈
- root 用户运行
- HTTP 明文（必须 HTTPS）

【强制】

- HTTPS / TLS 1.3
- 全链路加密
- 强密码策略（≥ 12 位，含大小写 + 数字 + 符号）
- 多因素认证（高敏角色）
- 异常登录告警

现在请生成完整安全合规实现。
```
