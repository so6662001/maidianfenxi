# 性能优化指南

## 一、性能目标速查

详见 [SLO 文档](../14-operations/04-slo-sli.md)。

## 二、后端 Java 优化

### JVM 调优

```bash
# 推荐 G1GC
-XX:+UseG1GC
-Xms2g -Xmx4g                     # 与容器 limits 匹配
-XX:MaxRAMPercentage=75.0         # 容器中用比例
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:+AlwaysPreTouch
-XX:+DisableExplicitGC

# 内存溢出快照
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dump/heap.hprof

# JFR（生产可开）
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=app.jfr

# 时区 + 字符集
-Duser.timezone=Asia/Shanghai
-Dfile.encoding=UTF-8

# Netty 内存池
-Dio.netty.allocator.type=pooled
-Dio.netty.leakDetection.level=advanced
```

### 线程池规范

```java
// 业务线程池（必须命名 + 拒绝策略）
@Bean("queryExecutor")
public ThreadPoolTaskExecutor queryExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(32);
    executor.setQueueCapacity(1000);
    executor.setKeepAliveSeconds(60);
    executor.setThreadNamePrefix("query-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(30);
    return executor;
}
```

**禁止**：
- `Executors.newCachedThreadPool()`（无界）
- `Executors.newFixedThreadPool()`（无界队列）
- 没有命名的线程池

### 数据库优化

#### 慢查询规范
- 所有 SQL 必须有索引
- 单表查询时间 < 100ms
- 复杂查询用物化视图
- 分页用游标，禁止 `OFFSET > 10000`

#### MyBatis 优化
```xml
<!-- 大结果集用 ResultHandler，避免内存撑爆 -->
<select id="exportLargeDataset" resultType="..." fetchSize="1000">
  ...
</select>
```

#### 连接池（Druid）
```yaml
druid:
  initial-size: 5
  min-idle: 10
  max-active: 50
  max-wait: 3000
  test-while-idle: true
  validation-query: SELECT 1
  validation-query-timeout: 1
```

### 缓存策略

#### 多级缓存

```
L1 Caffeine（本地，进程内，1 分钟）
   ↓ miss
L2 Redis（分布式，5-30 分钟）
   ↓ miss
L3 DB（持久层）
```

#### 缓存原则
- 高频读、低频写 → 缓存
- 数据变化通过事件清缓存（不靠 TTL）
- 大 Key（> 5MB）拆分或不缓存
- Hash / Set 元素 ≤ 5000
- 必设 TTL（最大 24h）
- 缓存穿透：空值缓存（短 TTL）
- 缓存雪崩：TTL 加随机
- 缓存击穿：分布式锁 + 单飞（singleflight）

#### Redisson 单飞模式

```java
public <T> T getWithSingleFlight(String key, Supplier<T> loader, Duration ttl) {
    T cached = (T) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;
    
    RLock lock = redisson.getLock("lock:" + key);
    try {
        lock.lock(10, TimeUnit.SECONDS);
        cached = (T) redisTemplate.opsForValue().get(key);  // 双检
        if (cached != null) return cached;
        
        T fresh = loader.get();
        redisTemplate.opsForValue().set(key, fresh, ttl);
        return fresh;
    } finally {
        lock.unlock();
    }
}
```

### ClickHouse 优化

```sql
-- 1. 必须分区过滤
SELECT ... WHERE event_time >= '2026-05-01' AND ...

-- 2. 物化视图常用聚合
CREATE MATERIALIZED VIEW mv_dau TO dws_dau AS
SELECT toDate(event_time) AS date, app_id, uniqState(user_id) AS dau
FROM dwd_event GROUP BY date, app_id;

-- 3. Skip Index 跳数索引
ALTER TABLE dwd_event ADD INDEX idx_event_code event_code TYPE bloom_filter GRANULARITY 4;

-- 4. JOIN 改为 IN（小表）
WHERE user_id IN (SELECT user_id FROM ...)

-- 5. 避免大表自 JOIN，用 window 函数

-- 6. 限制返回行数
SELECT ... LIMIT 1000

-- 7. SAMPLE 用于估算
SELECT count() FROM dwd_event SAMPLE 0.1 WHERE ...
```

### Spring Boot 启动优化

- Spring Boot 3.x AOT 编译（GraalVM Native）
- 懒加载（`spring.main.lazy-initialization=true`）
- 必要 Bean 提前加载

### HTTP 客户端

```java
@Bean
public OkHttpClient okHttpClient() {
    return new OkHttpClient.Builder()
        .connectionPool(new ConnectionPool(50, 5, TimeUnit.MINUTES))
        .connectTimeout(3, TimeUnit.SECONDS)
        .readTimeout(5, TimeUnit.SECONDS)
        .writeTimeout(5, TimeUnit.SECONDS)
        .retryOnConnectionFailure(true)
        .build();
}
```

## 三、前端优化

### 构建优化

```ts
// vite.config.ts
export default defineConfig({
  build: {
    target: 'esnext',
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-antd': ['ant-design-vue'],
          'vendor-echarts': ['echarts'],
        },
      },
    },
    chunkSizeWarningLimit: 500,
  },
  optimizeDeps: {
    include: ['vue', 'ant-design-vue'],
  },
});
```

### Bundle 优化

- 路由懒加载
- 组件按需引入（Ant Design Vue 自动）
- 图标按需引入
- ECharts 按图表类型引入
- 第三方大包 CDN（按需）

### 渲染优化

- 大列表虚拟滚动（vue-virtual-scroller）
- 图表数据 transformer 用 Worker
- 防抖节流（输入框、scroll）
- v-once / v-memo（不变内容）
- 异步组件（defineAsyncComponent）

### 网络优化

- HTTP/2 + Brotli
- 静态资源 CDN + 长缓存
- API 响应 gzip
- 关键 API 预请求
- Service Worker 缓存（PWA）

### 图表性能

```ts
// ECharts 大数据量
option = {
  series: [{
    type: 'scatter',
    progressive: 500,  // 渐进渲染
    progressiveThreshold: 5000,
    large: true,        // 大数据模式
    largeThreshold: 2000,
    sampling: 'lttb',  // 降采样
  }]
};
```

### 嵌入组件优化

- Tree-shaking（按需引入）
- 单组件 gzip ≤ 30KB
- 异步加载非关键组件
- 错误边界 + 降级渲染

## 四、SQL 性能优化

### 索引规范

- 高频查询字段必须有索引
- 联合索引顺序：选择性高 → 低
- 避免索引失效（前缀模糊 / 函数）
- 索引数量 ≤ 5 个/表
- 写多读少不加索引

### EXPLAIN 必读

```sql
EXPLAIN ANALYZE SELECT ...;
```

关注：
- type: 必须 ref / range / eq_ref
- key: 必须有索引
- rows: 越小越好
- Extra: 避免 Using filesort / Using temporary

### 查询优化清单

- [ ] 有 WHERE 过滤
- [ ] 有 LIMIT
- [ ] 用索引
- [ ] 避免 SELECT *
- [ ] 避免 JOIN > 3 表
- [ ] 避免子查询（用 JOIN 替代）
- [ ] 大表更新分批
- [ ] 历史数据归档

## 五、LLM 性能与成本优化

### 模型路由

```
意图分类（小模型 100ms）
├─ 简单（问候、改错）→ 小模型（GPT-4o-mini ¥0.001/k）
├─ 中等（写作、总结）→ 中模型（Claude Haiku ¥0.01/k）
└─ 复杂（推理、分析）→ 大模型（GPT-4 ¥0.1/k）
```

预计节省 30-50% 成本。

### Prompt 优化

- 系统 Prompt 简洁（避免冗余）
- Few-shot 示例精简（2-3 个）
- 输出格式强约束（JSON Schema）
- 思维链（CoT）只在必要时启用

### 缓存

```ts
// 语义缓存
const cacheKey = hash(normalize(prompt));
const cached = redis.get(`llm:cache:${cacheKey}`);
if (cached) return cached;

const result = await llm.call(prompt);
redis.set(`llm:cache:${cacheKey}`, result, 'EX', 3600);
```

预计命中率 20-30%。

### 流式响应

```java
return chatModel.streamCall(prompt)
    .map(chunk -> SseEvent.builder().data(chunk).build());
```

减少首字延迟，提升用户体验。

### 并发控制

- 单用户 QPS 限 5/s
- 单租户 QPS 限 100/s
- 全局并发 LLM 调用 ≤ 200

## 六、性能问题排查

### 高 CPU
```bash
# 1. 找进程
top -H -p <PID>

# 2. 转换线程 ID
printf "%x\n" <thread_id>

# 3. JStack 找堆栈
jstack <PID> | grep -A 30 <hex_thread_id>

# 4. Async Profiler 火焰图
./profiler.sh -d 30 -f flame.html <PID>
```

### 高内存
```bash
# 1. 堆 Dump
jmap -dump:format=b,file=heap.hprof <PID>

# 2. MAT 分析
# 找 dominator tree 看大对象

# 3. JFR 持续监控
```

### 高 GC
- 查 GC 日志：`-Xlog:gc*:file=gc.log`
- GC Easy 分析：https://gceasy.io
- 调整堆大小 / GC 算法

### 慢请求
- SkyWalking 链路追踪定位最慢环节
- DB 慢查询日志
- 缓存命中率

## 七、性能优化案例库

每个优化案例记录到决策追溯库：
- 问题描述
- 性能基线
- 优化方案
- 优化后效果
- 经验沉淀

## 八、性能预算

| 模块 | 单次请求最大 |
|---|---|
| API 响应 | 500ms / 5MB |
| SQL 查询 | 30s / 1GB 内存 |
| LLM 调用 | 30s / 8K Token |
| 触达发送 | 30s / 批 |
| 前端首屏资源 | 2MB |

超出预算需评审。
