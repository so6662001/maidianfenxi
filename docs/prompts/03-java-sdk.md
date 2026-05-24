# 提示词 03 · Java 数据上报 SDK

> 前置：`00-master-prompt.md`。参考：`docs/11-embedded-components/01-sdk-design.md`。

---

## 提示词正文

```
你是一名 SDK 开发专家。请开发 "analytics-java-sdk"。

【目标】
轻量、零侵入、线程安全的事件上报 SDK，供任意 Java 8+ 业务系统接入。

【两个产物】

1. analytics-java-sdk（核心，不依赖 Spring）
2. analytics-spring-boot-starter（Spring Boot 自动装配）

【依赖最小化】

仅允许：
- OkHttp 4.12
- Jackson 2.16
- SLF4J 2.0（不含具体实现）
- Chronicle Queue（可选，本地落盘）

【禁止依赖】

- Spring 任何版本
- Guava / Apache Commons / Hutool

【analytics-java-sdk 核心 API】

```java
public interface AnalyticsClient extends AutoCloseable {
    void track(String userId, String eventCode, Map<String, Object> properties);
    void track(String userId, String eventCode, Map<String, Object> properties, Long eventTime);
    void identify(String anonymousId, String userId, Map<String, Object> traits);
    void groupAssoc(String userId, String orgId);
    void setUserProperty(String userId, Map<String, Object> properties);
    void setUserOnce(String userId, Map<String, Object> properties);
    void flush();
    void close();
}

public class AnalyticsClientBuilder {
    public static AnalyticsClientBuilder newBuilder();
    public AnalyticsClientBuilder endpoint(String url);
    public AnalyticsClientBuilder appKey(String key);
    public AnalyticsClientBuilder appSecret(String secret);
    public AnalyticsClientBuilder batchSize(int size);
    public AnalyticsClientBuilder flushIntervalMs(long ms);
    public AnalyticsClientBuilder maxQueueSize(int size);
    public AnalyticsClientBuilder enableDiskBuffer(boolean enable);
    public AnalyticsClientBuilder diskBufferPath(String path);
    public AnalyticsClientBuilder httpTimeoutMs(long ms);
    public AnalyticsClientBuilder httpMaxRetries(int retries);
    public AnalyticsClientBuilder debug(boolean debug);
    public AnalyticsClient build();
}
```

【核心实现要点】

1. **异步批量上报**
   - 内部 LinkedBlockingQueue（max-queue-size 默认 100k）
   - 单消费线程：定时（flush-interval-ms）或满批（batch-size）触发
   - 满时按策略丢弃（dropOldest / dropNewest 可配，默认 dropOldest 并 log warn）

2. **HTTP 调用**
   - OkHttp 单实例（连接池复用）
   - 超时：connect 3s / read 5s
   - 失败重试：指数退避（1s/2s/4s），最多 3 次
   - 仍失败 → 写本地磁盘队列

3. **HMAC 签名**
   - 算法：HMAC-SHA256
   - sign_string = METHOD + "\n" + PATH + "\n" + TIMESTAMP + "\n" + NONCE + "\n" + MD5(BODY)
   - 签名结果 Base64
   - Headers: X-App-Key / X-Timestamp / X-Nonce / X-Signature

4. **本地磁盘队列**
   - 基于 Chronicle Queue 或简单 RandomAccessFile + 索引
   - 最大磁盘占用配置（默认 1GB），超出 FIFO 丢弃
   - 启动时自动加载并重发

5. **优雅关闭**
   - Runtime.getRuntime().addShutdownHook
   - flush() 强制发送队列
   - close() 关闭线程池 + 释放资源
   - 等待最多 5s

6. **错误处理**
   - 所有异常捕获 + log warn，不抛出业务方
   - debug 模式打详细日志

【analytics-spring-boot-starter 实现】

1. AnalyticsProperties（@ConfigurationProperties("analytics")）
2. AnalyticsAutoConfiguration（@AutoConfiguration + @ConditionalOnProperty）
   - 创建 AnalyticsClient Bean
   - 注册 AnalyticsAspect（@Aspect 实现 @Track 注解）
3. @EnableAnalytics（@Import 触发自动配置）
4. @Track 注解 + AnalyticsAspect
   - SpEL 解析 event/props
   - 自动获取 userId（从 SecurityContext 或 ThreadLocal）
5. resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

【@Track 注解定义】

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Track {
    String event();
    String props() default "";              // SpEL 表达式
    String userId() default "";             // SpEL，默认从 SecurityContext
    boolean trackException() default false; // 异常也上报
    boolean async() default true;
}
```

【测试要求】

1. 单元测试覆盖率 ≥ 80%
2. 必测场景：
   - 批量上报正常
   - 队列满丢弃
   - 网络失败重试
   - 本地落盘 + 重启加载
   - HMAC 签名正确性
   - 优雅关闭
   - 多线程并发
3. 集成测试：用 MockWebServer 模拟 Collector

【性能要求】

- 单 JVM 支持 ≥ 10 万 events/s 入队（业务方调用 track 不阻塞）
- track() 调用延迟 P99 < 1ms
- 内存占用 ≤ 100MB（队列 + 元数据）

【交付物】

- 完整 Maven 模块代码
- README.md（含接入指南：Spring Boot 与非 Spring 两种）
- CHANGELOG.md
- 集成示例工程：analytics-sdk-demo-springboot / analytics-sdk-demo-plain
- Javadoc（mvn javadoc:javadoc 通过）
- JaCoCo 覆盖率报告
- 性能压测脚本（JMH）+ 报告

【禁止】

- 禁止反射魔改业务对象
- 禁止吞掉异常不打日志
- 禁止在主线程做网络 IO
- 禁止依赖 Spring（核心 SDK）
- 禁止 System.out / printStackTrace
- 禁止 AppSecret 出现在任何日志

【安全】

- AppSecret 仅在 builder 时传入，不暴露 getter
- log 输出脱敏（自动检测并替换 secret-like 字符串）
- 大 properties（> 64KB）自动截断 + warn

现在请生成完整代码 + 测试 + 文档。
```
