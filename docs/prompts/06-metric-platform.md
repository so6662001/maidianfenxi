# 提示词 06 · 指标平台服务

> 前置：`00-master-prompt.md`。参考：`docs/03-data-models/04-metrics-dictionary.md` + `05-metrics-sql.md`。

---

## 提示词正文

```
你是数据中台架构师。请开发 analytics-metric 服务。

【目标】
"指标定义—计算—查询"全生命周期管理。所有上层应用只消费指标，不直接写 SQL。

【技术】

- Spring Boot 3.2
- Apache Calcite（SQL 解析 + 优化）
- ClickHouse JDBC
- MySQL + MyBatis-Plus
- Redis + Redisson
- Caffeine

【核心模块】

```
analytics-metric/
├── controller/
│   ├── MetricController          # 查询 API
│   └── MetricMetaController      # 元数据管理 API
├── service/
│   ├── MetricMetaService          # 指标元数据
│   ├── MetricQueryService         # 查询执行
│   ├── MetricCacheService         # 缓存管理
│   └── MetricAuditService         # 审计
├── dsl/                           # JSON DSL → SQL
│   ├── DslParser
│   ├── DslValidator
│   └── SqlBuilder（Calcite）
├── executor/
│   ├── ClickHouseExecutor
│   ├── MaterializedViewRouter
│   └── QueryTimeoutGuard
├── repository/
│   ├── MetricMetaMapper
│   └── MetricAuditMapper
└── domain/
    └── entity/
```

【指标元模型】

```yaml
原子指标:
  - code: M-SE-001
  - name: NRR
  - category: 扩张续约
  - business_def: 净收入留存
  - calc_formula: (期初ARR + 扩张 - 收缩 - 流失) / 期初ARR
  - sql_template: <SQL>
  - dimensions: [product, industry, region, ...]
  - threshold: { green: 110, yellow: 100, red: 90 }
  - update_freq: daily
  - owner: <user_id>
  
派生指标:
  - 引用原子指标 + 公式
  
复合指标:
  - 多原子指标加权
```

【查询 DSL（JSON）】

```json
{
  "metric": "M-SE-001",
  "dimensions": ["product", "industry"],
  "filters": [
    { "field": "product", "op": "in", "value": ["saas-invoice"] }
  ],
  "dateRange": { "start": "2026-05-01", "end": "2026-05-23" },
  "granularity": "day",
  "compare": "mom",
  "limit": 100,
  "cursor": null
}
```

【查询执行流程】

```
DSL 接收
   ↓
DSL Validator（字段权限 / 维度合法性）
   ↓
权限注入（tenant_id + 行级权限）
   ↓
缓存查询（Redis，key = hash(DSL + 用户上下文)）
   ├─ 命中 → 返回
   └─ 未命中
      ↓
   物化视图路由（自动选择最匹配的预聚合表）
   ├─ 命中物化 → 改写 SQL
   └─ 未命中 → 查明细表 dwd_event
      ↓
   Calcite 解析 + 优化
      ↓
   ClickHouse 执行
      ├─ 慢查询保护（30s 超时 kill）
      └─ 结果返回
      ↓
   写缓存（TTL 按指标 update_freq 配置）
      ↓
   审计日志（ES）
      ↓
   返回响应
```

【物化视图路由】

- 维护"指标 → 物化视图"映射表
- 物化视图字段集 ⊇ 查询字段集 → 路由
- 优先选择数据量最小的物化

【缓存策略】

- 实时指标：TTL 1 分钟
- 日报：TTL 1 小时
- 周/月报：TTL 24 小时
- 缓存 key 含用户身份哈希（确保权限隔离）
- Redis 大 Key 监控（> 100KB 拆分）

【慢查询保护】

```java
@Async("queryExecutor")
public CompletableFuture<MetricResult> execute(...) {
    return CompletableFuture
        .supplyAsync(() -> clickHouseClient.query(...), queryExecutor)
        .orTimeout(30, TimeUnit.SECONDS)
        .exceptionally(ex -> {
            log.warn("Query timeout, kill query");
            clickHouseClient.killQuery(queryId);
            throw new BusinessException(120002, "查询超时");
        });
}
```

【行级权限注入】

```java
// 所有查询自动注入
WHERE tenant_id = :currentTenantId
  AND (:isAdmin OR org_id IN (:userOrgIds))
  AND (:isAdmin OR product_line IN (:userProductLines))
```

【API】

```
POST /api/metric/query              # 查询指标
GET  /api/metric/{code}             # 指标元数据
GET  /api/metric/list               # 指标列表
POST /api/meta/metric               # 创建指标（评审权限）
PUT  /api/meta/metric/{code}        # 更新指标（评审权限）
DELETE /api/meta/metric/{code}      # 下线指标
GET  /api/meta/metric/audit         # 查询审计日志
```

【审计】

每次查询记录：
- traceId / userId / metric / DSL / 行级权限上下文
- 执行时长 / 缓存命中 / 行数
- 异常（如有）

存 Elasticsearch，保留 180 天。

【性能要求】

- 简单指标查询 P95 ≤ 500ms（含缓存）
- 复杂即席查询 P95 ≤ 8s
- 缓存命中率 ≥ 70%
- 物化视图命中率 ≥ 60%

【验证】

1. 指标 DSL 解析正确率 100%
2. 物化路由准确率
3. 权限隔离测试（不可越权访问其他租户/产品线数据）
4. 慢查询 kill 测试
5. 大并发查询不雪崩（限流 + 熔断）

【交付物】

- 完整服务代码 + 单测（≥ 80% 覆盖）
- DSL 规范文档
- 指标接入流程文档
- Prometheus 监控指标
- 性能压测报告

【禁止】

- 禁止用户直接传 SQL
- 禁止跳过权限校验
- 禁止无缓存的高频查询
- 禁止 SELECT *
- 禁止字符串拼 SQL

【安全】

- 所有用户输入校验
- 行级权限强制
- 审计完整
- 慢查询自动 kill

现在请生成完整服务代码。
```
