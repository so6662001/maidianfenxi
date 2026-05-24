# SLO / SLI / Error Budget 定义

## 一、概念

| 概念 | 定义 |
|---|---|
| **SLI**（Service Level Indicator） | 度量服务质量的指标，如可用性、延迟 |
| **SLO**（Service Level Objective） | 服务水平目标，给出 SLI 的目标值 |
| **SLA**（Service Level Agreement） | 对外承诺（带违约赔偿） |
| **Error Budget** | 1 - SLO，允许的"故障预算" |

## 二、SLO 总览

| 服务 | 可用性 SLO | 延迟 SLO（P95） | Error Budget（月） |
|---|---|---|---|
| analytics-gateway | 99.95% | < 100ms | 21.6 min |
| analytics-collector | 99.99% | < 50ms | 4.3 min |
| analytics-auth | 99.99% | < 100ms | 4.3 min |
| analytics-metric | 99.9% | < 500ms | 43.2 min |
| analytics-segment | 99.9% | < 2s | 43.2 min |
| analytics-reach | 99.9% | < 1s | 43.2 min |
| analytics-experiment | 99.95% | < 50ms（variant） | 21.6 min |
| analytics-report | 99.9% | < 3s | 43.2 min |
| analytics-decision | 99.9% | < 1s | 43.2 min |
| analytics-assistant | 99.5% | < 8s（含 LLM） | 3.6 h |
| analytics-insight | 99.5% | — | 3.6 h |
| analytics-openapi | 99.95% | < 500ms | 21.6 min |
| 数据通路（Kafka→ClickHouse） | 99.9%（成功率） | 数据新鲜度 < 4h（P99） | — |

## 三、详细 SLI/SLO（按服务）

### analytics-gateway

| SLI | 公式 | SLO |
|---|---|---|
| 可用性 | `sum(http_requests{status!~"5.."}) / sum(http_requests)` | ≥ 99.95% |
| 延迟 | `histogram_quantile(0.95, http_request_duration)` | < 100ms |
| 鉴权失败率 | `auth_failed / auth_total` | < 0.1% |

### analytics-collector

| SLI | 公式 | SLO |
|---|---|---|
| 可用性 | 同上 | ≥ 99.99% |
| 接收延迟 | P99 | < 50ms |
| 数据丢失率 | `(received - kafka_written) / received` | < 0.001% |
| 校验失败率 | `invalid / received` | < 1% |

### analytics-metric

| SLI | 公式 | SLO |
|---|---|---|
| 可用性 | 同上 | ≥ 99.9% |
| 查询延迟 P95 | — | < 500ms |
| 查询延迟 P99 | — | < 3s |
| 缓存命中率 | `cache_hit / (cache_hit + cache_miss)` | ≥ 70% |
| 慢查询率 | `slow_query / total_query` | < 1% |

### analytics-assistant

| SLI | 公式 | SLO |
|---|---|---|
| 可用性 | 同上 | ≥ 99.5% |
| 流式首字延迟 | — | < 800ms |
| 简单查询响应 | — | < 2s |
| 复杂分析响应 | P95 | < 8s |
| 用户满意度 | 👍 / (👍+👎) | ≥ 70% |
| Function 调用成功率 | — | ≥ 99% |
| LLM 调用成功率 | — | ≥ 99% |
| Token 成本 / 会话 | — | 按场景预算 |

### analytics-reach

| SLI | 公式 | SLO |
|---|---|---|
| 任务调度延迟 | — | < 5s |
| 通道发送成功率 | — | ≥ 99% |
| 通道延迟 P95 | — | 站内 < 1s / 邮件 < 30s / 短信 < 5s |
| 退订处理率 | — | 100% |

### 数据通路

| SLI | 公式 | SLO |
|---|---|---|
| 数据新鲜度（实时） | `now() - max(event_time in ClickHouse)` | P99 < 5 min |
| 数据新鲜度（T+0） | — | P99 < 1 h |
| 数据新鲜度（T+1） | — | P99 < 4 h |
| Kafka 端到端延迟 | — | P99 < 1 min |
| 数据完整性 | `clickhouse_count / kafka_count` | ≥ 99.99% |

## 四、Error Budget 政策

```
月度 Error Budget 用尽 → 冻结新功能发布，专注稳定性
月度 Error Budget 50% 用尽 → 警告 + 评估
月度 Error Budget 25% 用尽 → 通知 oncall + 监控
```

### Error Budget Burn Rate 告警

```yaml
# 1h 内消耗 24h 预算 → P0
- alert: ErrorBudgetBurnFast
  expr: |
    (1 - sum(rate(http_requests{status!~"5.."}[1h])) / sum(rate(http_requests[1h])))
    > (1 - 0.999) * 24
  severity: P0

# 6h 内消耗 7d 预算 → P1
- alert: ErrorBudgetBurnSlow
  expr: |
    (1 - sum(rate(http_requests{status!~"5.."}[6h])) / sum(rate(http_requests[6h])))
    > (1 - 0.999) * 4.5
  severity: P1
```

## 五、SLO 看板（Grafana）

每个服务一个看板：
- 当前可用性 vs SLO（环形）
- 30 天可用性趋势
- 当前 Error Budget 剩余 + Burn Rate
- 延迟 P50/P95/P99 趋势
- 流量 + 错误率
- 影响最大的 Top 5 错误码

## 六、SLA（对外承诺）

| 类别 | SLA | 违约赔偿 |
|---|---|---|
| **核心服务可用性** | 99.9% / 月 | 按比例返还会员费 |
| **数据新鲜度** | T+1 完成时间 06:00 | 延迟通报 |
| **OpenAPI 可用性** | 99.95% | 同上 |
| **数据安全** | 100% 不泄露 | 严格责任 |

SLA 与公司法务确认后，写入业务方接入合同。

## 七、Error Budget 使用案例

```
案例 1：连续 3 周可用性 99.85%，月度预算用尽
  → 冻结新功能 2 周
  → 专项优化稳定性
  → 加强混沌测试

案例 2：因发版导致 30 分钟故障，消耗预算 70%
  → 评估发布流程
  → 强化灰度发布机制
  → 加严回滚 SLA
```

## 八、SLO 评审

- **月度评审**：所有服务 SLO 达成情况
- **季度评审**：调整 SLO 目标（业务变化）
- **年度评审**：SLO 战略调整

评审输出归档到决策追溯库。
