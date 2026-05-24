# 提示词 01 · 后端 Spring Boot 多模块项目初始化

> 前置：先阅读 `00-master-prompt.md`。

---

## 提示词正文

```
你是一名资深 Java 架构师。请基于以下规约创建一个名为 "analytics-platform" 的多模块 Maven 项目。

【目标】
生成可直接 `mvn clean package -DskipTests` 编译通过、`mvn spring-boot:run` 启动的项目骨架。

【上下文】
本项目是"用户分析中台 + 决策操作系统"的后端骨架。MVP 阶段先合并为 4 个服务，跑顺再细拆。

【模块结构】

analytics-platform/                              # 父项目
├── pom.xml                                      # 父 pom（依赖管理）
├── analytics-common/                            # 通用模块（工具/常量/异常/响应）
│   ├── pom.xml
│   └── src/main/java/com/yourcompany/analytics/common/
│       ├── exception/  (BusinessException, GlobalExceptionHandler)
│       ├── response/   (Result<T>, ResultCode, PageResult)
│       ├── utils/      (DateUtil, JsonUtil, IdGenerator-Snowflake, HmacUtil)
│       ├── constant/   
│       └── config/     (基础配置类)
├── analytics-api/                               # 对外 DTO + Feign 接口
│   ├── pom.xml
│   └── src/main/java/com/yourcompany/analytics/api/
│       └── dto/
├── analytics-starter/                           # 自定义 starter 集合
│   └── pom.xml
├── analytics-gateway/                           # 网关服务（8000）
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/main/{java,resources}/
├── analytics-data/                              # 数据服务（8001-8004 合并）
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/main/{java,resources}/
├── analytics-app/                               # 应用服务（8005-8010 合并）
├── analytics-assistant/                         # AI 助理 + 洞察
└── helm/                                        # K8s 部署
    └── analytics-platform/

【父 pom.xml 关键内容】

- spring-boot-starter-parent 3.2.5
- spring-cloud-dependencies 2023.0.1
- spring-cloud-alibaba-dependencies 2023.0.1.0
- Java 17
- 字符编码 UTF-8
- Maven 编译插件 + Spring Boot 插件
- Spotless 代码格式化插件（阿里规约）
- CheckStyle Maven Plugin（阿里规约 ruleset）
- JaCoCo Maven Plugin（覆盖率门禁 60%）
- 子模块依赖版本统一在 dependencyManagement 中锁定

【关键依赖锁定】

| 依赖 | 版本 |
|---|---|
| Spring Boot | 3.2.5 |
| Spring Cloud | 2023.0.1 |
| Spring Cloud Alibaba | 2023.0.1.0 |
| MyBatis-Plus | 3.5.5 |
| Dynamic Datasource | 4.3.0 |
| Druid | 1.2.21 |
| Redisson | 3.27.2 |
| Lettuce | 6.3.2.RELEASE |
| Spring Kafka | 3.1.x |
| ClickHouse JDBC | 0.6.0 |
| MySQL Connector/J | 8.3.0 |
| Jackson | 2.16.x |
| OkHttp | 4.12.0 |
| Lombok | 1.18.30 |
| Hibernate Validator | 8.0.1 |
| SpringDoc OpenAPI | 2.3.0 |
| Logback | 1.4.14 |
| SLF4J | 2.0.x |
| Caffeine | 3.1.8 |
| Hutool | 5.8.25 |
| ip2region | 2.7.0 |

【analytics-common 必须实现】

1. Result<T> 统一响应（含 code/message/data/traceId/requestId）
2. ResultCode 枚举（成功/失败/各类业务码）
3. BusinessException 业务异常（含错误码 + 国际化消息）
4. GlobalExceptionHandler @RestControllerAdvice（全局异常处理）
5. IdGenerator 雪花算法（机器 ID 从配置读取）
6. JsonUtil 基于 Jackson（统一序列化策略：Long 转字符串、日期格式 ISO8601）
7. HmacUtil HMAC-SHA256 签名 + 验签
8. DateUtil 时间工具（时区统一 UTC+8 配置化）
9. TraceIdFilter 注入 TraceId 到 MDC（与 Sleuth/SkyWalking 兼容）
10. ApiResponseAdvice 自动包装 Controller 返回值为 Result<T>

【analytics-gateway 必须实现】

1. Spring Cloud Gateway 路由配置（dev/prod 不同）
2. JwtAuthFilter（验证 JWT，注入用户信息到 Header）
3. AppKeySignFilter（OpenAPI HMAC 签名验证）
4. RateLimitFilter（基于 Sentinel）
5. CorsConfig（统一 CORS）
6. TraceIdInjectFilter
7. /actuator/health + /actuator/prometheus

【每个服务必须实现】

1. Application 主类 + @EnableDiscoveryClient + @EnableFeignClients
2. application-{dev,test,staging,prod}.yml
3. bootstrap.yml（Nacos 配置）
4. SpringDoc OpenAPI 配置（/swagger-ui.html）
5. MyBatis-Plus 配置（自动填充 gmt_create/gmt_modified/creator_id/modifier_id）
6. Redis 配置（Redisson）
7. Druid 数据源 + 监控
8. 全局异常处理
9. Swagger 分组配置
10. Dockerfile（multi-stage build，基于 eclipse-temurin:17-jre-jammy）
11. Helm Chart 模板（资源限制 / HPA / PDB / ConfigMap）

【日志规范】

logback-spring.xml：
- 输出 JSON 格式
- 含字段：timestamp / level / service / traceId / spanId / message / mdc / exception
- 控制台 + 文件 + 异步 Appender
- 不同环境不同级别（dev DEBUG / prod INFO）

【数据库迁移】

使用 Flyway，目录 `src/main/resources/db/migration/`，命名 `V1.0.0__init.sql`。

【验收标准】

1. `mvn clean package -DskipTests` 一次通过
2. 任意服务可独立启动，注册到 Nacos
3. 访问 `/actuator/health` 返回 `{"status":"UP"}`
4. 访问 `/swagger-ui.html` 可见 API 文档
5. CheckStyle 0 错误
6. Spotless format 通过
7. 单测可运行（即使为空）
8. Dockerfile 可构建镜像 < 250MB
9. Helm Chart `helm install --dry-run` 通过
10. 所有 yml 配置文件含完整注释

【输出格式】

按文件组织，每个文件用 ```{语言}:{完整路径}``` 包裹。
输出顺序：
1. 父 pom.xml
2. analytics-common/
3. analytics-api/
4. analytics-starter/
5. analytics-gateway/
6. analytics-data/
7. analytics-app/
8. analytics-assistant/
9. helm/
10. README.md + .gitignore + .editorconfig

【禁止】

- 禁止使用已废弃 API（如 javax.* 替换为 jakarta.*）
- 禁止使用 @Autowired 字段注入（用构造函数注入 + Lombok @RequiredArgsConstructor）
- 禁止留 TODO / 注释代码
- 禁止 System.out / printStackTrace
- 禁止硬编码配置（必须配置化）
- 禁止使用过期版本（如 Spring Boot 2.x）

现在请开始生成项目骨架。
```
