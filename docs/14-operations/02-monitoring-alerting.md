# 监控与告警

## 一、监控体系

```
┌─────────────────────────────────────────────────────────────┐
│  L1 业务监控（北极星指标、Playbook ROI、决策达成率）          │
├─────────────────────────────────────────────────────────────┤
│  L2 应用监控（服务可用性、QPS、延迟、错误率）                 │
├─────────────────────────────────────────────────────────────┤
│  L3 中间件监控（MySQL/Redis/Kafka/ClickHouse 健康度）         │
├─────────────────────────────────────────────────────────────┤
│  L4 基础设施监控（K8s/节点/网络）                             │
└─────────────────────────────────────────────────────────────┘
```

## 二、监控工具栈

| 工具 | 用途 |
|---|---|
| **Prometheus** | Metrics 抓取与存储 |
| **Grafana** | 可视化看板 |
| **AlertManager** | 告警路由与去重 |
| **SkyWalking** | APM + 链路追踪 |
| **ELK / Loki** | 日志聚合与查询 |
| **kube-state-metrics** | K8s 资源监控 |
| **node-exporter** | 节点资源监控 |
| **blackbox-exporter** | 外部探测 |

## 三、关键指标清单

### L1 业务指标（Prometheus + Grafana）

| 指标 | 类型 | 单位 |
|---|---|---|
| `analytics_active_orgs_total` | Gauge | 个 |
| `analytics_dau_total` | Gauge | 个 |
| `analytics_decision_registered_total` | Counter | 个 |
| `analytics_decision_review_completion_rate` | Gauge | % |
| `analytics_playbook_executions_total{playbook}` | Counter | 次 |
| `analytics_playbook_roi{playbook}` | Gauge | x |
| `analytics_assistant_conversations_total` | Counter | 次 |
| `analytics_assistant_thumbsup_rate` | Gauge | % |
| `analytics_segment_active_total` | Gauge | 个 |
| `analytics_journey_running_total` | Gauge | 个 |

### L2 应用指标（每服务）

| 指标 | 类型 |
|---|---|
| `http_requests_total{service, method, path, status}` | Counter |
| `http_request_duration_seconds{service, method, path}` | Histogram |
| `http_requests_in_flight{service}` | Gauge |
| `jvm_memory_used_bytes{area}` | Gauge |
| `jvm_gc_pause_seconds{type}` | Histogram |
| `jvm_threads_live` | Gauge |
| `business_errors_total{service, code}` | Counter |
| `cache_hit_total{cache_name}` | Counter |
| `cache_miss_total{cache_name}` | Counter |

### 服务专用指标

#### Collector
- `collector_events_received_total{app_id}`
- `collector_events_dropped_total{app_id, reason}`
- `collector_events_invalid_total{app_id, reason}`
- `collector_kafka_send_duration_seconds`

#### Metric
- `metric_query_total{metric_code, cache_hit}`
- `metric_query_duration_seconds{metric_code}`
- `metric_slow_query_killed_total`
- `metric_materialized_view_hit_rate`

#### Assistant
- `assistant_chat_total{model}`
- `assistant_chat_duration_seconds{step}`
- `assistant_tool_calls_total{tool, status}`
- `assistant_llm_tokens_total{model, direction}`
- `assistant_llm_cost_yuan{model}`
- `assistant_user_feedback_total{type}`

#### Reach
- `reach_tasks_total{channel, status}`
- `reach_task_duration_seconds{channel}`
- `reach_delivery_rate{channel}`
- `reach_fatigue_blocked_total`

### L3 中间件指标

- MySQL: 连接数 / QPS / 慢查询数 / 复制延迟 / Buffer Pool 命中率
- Redis: QPS / 内存使用 / 连接数 / 命中率 / 大 Key
- Kafka: Producer/Consumer 速率 / Lag / 分区均衡 / ISR 副本
- ClickHouse: 查询 QPS / Merge 健康度 / 副本同步延迟

### L4 基础设施指标

- 节点：CPU / 内存 / 磁盘 IOPS / 网络
- K8s Pod：CPU / 内存 / 重启次数 / OOMKilled
- 网络：入出流量 / 延迟 / 丢包率

## 四、告警规则

### 告警分级

| 级别 | 响应时间 | 通道 | 升级 |
|---|---|---|---|
| **P0 灾难** | 立即 | 电话 + SMS + 企微 | L3 oncall + 全员 |
| **P1 严重** | 10 min | 企微 + 邮件 | L2 oncall |
| **P2 警告** | 1 h | 企微 | L1 oncall |
| **P3 信息** | 工作日 | 邮件日报 | — |

### 关键告警规则

```yaml
groups:
- name: analytics-platform-critical
  rules:
  
  # ===== P0 灾难 =====
  - alert: ServiceDown
    expr: up{job=~"analytics-.*"} == 0
    for: 2m
    labels: { severity: P0 }
    annotations:
      summary: "服务 {{ $labels.job }} 宕机"
  
  - alert: AllReplicaDown
    expr: kube_deployment_status_replicas_available{namespace="analytics-prod"} == 0
    for: 1m
    labels: { severity: P0 }
  
  - alert: DatabaseDown
    expr: mysql_up == 0 or clickhouse_up == 0
    for: 1m
    labels: { severity: P0 }
  
  - alert: AssistantSecretLeakDetected
    expr: increase(assistant_pii_leak_total[5m]) > 0
    labels: { severity: P0 }
  
  # ===== P1 严重 =====
  - alert: HighErrorRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
      / sum(rate(http_requests_total[5m])) by (service) > 0.05
    for: 3m
    labels: { severity: P1 }
  
  - alert: HighLatencyP95
    expr: histogram_quantile(0.95, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))) > 5
    for: 5m
    labels: { severity: P1 }
  
  - alert: KafkaLagHigh
    expr: kafka_consumer_lag > 100000
    for: 5m
    labels: { severity: P1 }
  
  - alert: PodCrashLoop
    expr: rate(kube_pod_container_status_restarts_total{namespace="analytics-prod"}[5m]) > 0
    for: 5m
    labels: { severity: P1 }
  
  - alert: DiskSpaceLow
    expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.10
    for: 10m
    labels: { severity: P1 }
  
  - alert: LLMCostSpike
    expr: |
      sum(rate(assistant_llm_cost_yuan[1h])) >
      avg_over_time(sum(rate(assistant_llm_cost_yuan[1h]))[7d:1h]) * 2
    for: 30m
    labels: { severity: P1 }
    annotations:
      summary: "LLM 成本异常飙升"
  
  # ===== P2 警告 =====
  - alert: CacheHitRateLow
    expr: |
      sum(rate(cache_hit_total[10m])) / 
      sum(rate(cache_hit_total[10m]) + rate(cache_miss_total[10m])) < 0.7
    for: 30m
    labels: { severity: P2 }
  
  - alert: SlowQueryHigh
    expr: rate(metric_slow_query_killed_total[10m]) > 0.1
    for: 30m
    labels: { severity: P2 }
  
  - alert: BatchJobDelayed
    expr: time() - xxljob_last_success_timestamp > 3600
    for: 30m
    labels: { severity: P2 }
  
  # ===== 业务告警 =====
  - alert: DauDrop
    expr: |
      (analytics_dau_total - analytics_dau_total offset 1d) / analytics_dau_total offset 1d < -0.20
    for: 1h
    labels: { severity: P1 }
    annotations:
      summary: "DAU 同比下降 > 20%"
  
  - alert: NoDecisionForLongTime
    expr: time() - analytics_last_decision_timestamp > 86400 * 3
    labels: { severity: P2 }
    annotations:
      summary: "≥ 3 天无决策登记"
```

## 五、告警通道路由

```yaml
# alertmanager.yml
route:
  receiver: default
  group_by: [alertname, severity, service]
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match: { severity: P0 }
    receiver: critical-team
    group_wait: 0s
    repeat_interval: 30m
  - match: { severity: P1 }
    receiver: oncall-team
    repeat_interval: 1h
  - match: { severity: P2 }
    receiver: dev-team

receivers:
- name: critical-team
  webhook_configs:
  - url: https://feishu-bot.../critical
  pagerduty_configs:
  - service_key: ...
  
- name: oncall-team
  webhook_configs:
  - url: https://feishu-bot.../oncall
  email_configs:
  - to: oncall@yourcompany.com

- name: dev-team
  webhook_configs:
  - url: https://feishu-bot.../dev
```

## 六、Grafana 看板

| 看板 | 内容 |
|---|---|
| **业务总览** | L1 业务指标 + 业务大盘 |
| **服务总览** | 所有服务的 QPS / 延迟 / 错误率 |
| **服务详情**（每服务一个） | JVM / 业务指标 / 上下游依赖 |
| **数据库** | MySQL / Redis / ClickHouse 健康 |
| **Kafka** | 生产/消费/Lag |
| **K8s 资源** | Pod / Node 资源 |
| **链路追踪** | SkyWalking 拓扑 |
| **AI 助理** | 对话 / Token / 成本 / 满意度 |
| **Playbook** | 触达 / ROI / 异动 |
| **决策追溯** | 登记数 / 复盘 / 案例库 |

每个看板必有：
- 时间筛选
- 环境/服务筛选
- 同环比对比
- 一键跳转明细

## 七、日志查询规范

### 统一日志格式（JSON）

```json
{
  "timestamp": "2026-05-24T10:00:00.123+08:00",
  "level": "INFO",
  "service": "analytics-metric",
  "traceId": "abc123",
  "spanId": "def456",
  "userId": "u-001",
  "tenantId": "t-001",
  "logger": "com.yourcompany.analytics.metric.MetricService",
  "thread": "http-nio-8004-exec-1",
  "message": "Query metric M-SE-001 success",
  "duration_ms": 234,
  "metric_code": "M-SE-001",
  "mdc": {...},
  "exception": null
}
```

### Kibana 查询示例

```
# 查特定用户的所有操作
userId: "u-001" AND timestamp:[now-1h TO now]

# 查链路全貌
traceId: "abc123"

# 查特定接口慢查询
service: "analytics-metric" AND duration_ms:>5000

# 查异常
level: "ERROR" AND service:"analytics-*"
```

## 八、值班工具

- **PagerDuty / OpsGenie**：告警接收 + 排班
- **企微/钉钉机器人**：日常通知
- **Runbook**：每个告警附带处理 SOP 链接（指向 03-troubleshooting.md）
- **复盘工具**：故障复盘模板自动归档到决策追溯库

## 九、监控数据保留

| 数据 | 保留 |
|---|---|
| Prometheus 短期（高分辨率） | 15 天 |
| Prometheus 长期（聚合） | 1 年（Thanos / VictoriaMetrics） |
| 日志（热） | 30 天 |
| 日志（冷归档） | 1 年 |
| 链路追踪 | 7 天 |
| 审计日志 | 180 天 |
