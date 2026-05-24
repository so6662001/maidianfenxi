# 提示词 05 · 数据采集服务 Collector

> 前置：`00-master-prompt.md`。参考：`docs/02-architecture/04-data-flow.md`。

---

## 提示词正文

```
你是高并发后端工程师。请开发 analytics-collector 服务。

【目标】承接所有客户端事件上报，做轻量校验+丰富后入 Kafka，单 Pod 4C8G ≥ 1 万 events/s。

【技术】

- Spring Boot 3.2 + WebFlux（响应式，避免线程阻塞）
- Reactor Kafka
- Caffeine（本地缓存）
- Redis（鉴权 + 元数据二级缓存）
- ip2region（IP 解析）
- ua_parser（UA 解析）

【接口】

POST /v1/collect/track     单条/批量事件
POST /v1/collect/identify  身份关联

请求体 / 响应详见 docs/12-api-spec/。

【处理流程】

```
请求进入
   ↓
鉴权：AppKey + HMAC 签名（Caffeine 缓存命中率 > 99%）
   ├─ Token 校验失败 → 401
   └─ 限流 Sentinel 触发 → 429 + Retry-After
   ↓
请求体反序列化（Jackson 流式）
   ↓
事件批量校验：
   ├─ 必填字段（event_code、user_id 或 device_id、event_time）
   ├─ 事件元数据存在性校验（Caffeine → Redis → meta 服务 fallback）
   └─ 时间漂移校验（与服务端时间相差 > 7 天进异常队列）
   ↓
字段丰富：
   ├─ server_time（当前）
   ├─ event_id（雪花）
   ├─ tenant_id、app_id
   ├─ IP → geo（ip2region）
   └─ UA → 浏览器/OS
   ↓
异步发 Kafka：events_raw（按 app_id 分区）
   ├─ 成功 → 返回 ACK
   └─ 失败 → 重试 3 次 → DLQ
   ↓
返回 { code: 0, accepted: N, traceId }
```

【关键设计】

1. **完全无状态**：可水平扩缩
2. **WebFlux 响应式**：避免线程阻塞，单 Pod 高 QPS
3. **不写数据库**：只入 Kafka，避免数据库成为瓶颈
4. **Sentinel 限流**：按 AppKey + 接口 双维度
5. **请求体大小限制**：单批次 ≤ 1MB，批量事件 ≤ 1000 条
6. **背压控制**：Kafka 发送失败时背压到客户端

【鉴权】

- HMAC-SHA256
- Token 缓存：Caffeine（1 分钟）+ Redis（5 分钟）
- Nonce 防重放：Redis 5 分钟窗口

【监控】

暴露 Prometheus 指标：
- collector_events_received_total{app_id}
- collector_events_dropped_total{app_id, reason}
- collector_events_invalid_total{app_id, reason}
- collector_request_duration_seconds（histogram）
- collector_kafka_lag

【告警】

- 错误率 > 5% 持续 1min
- P99 延迟 > 50ms 持续 5min
- Kafka 堆积 > 100k
- DLQ 增量异常

【灰度】

- 按 AppKey 染色路由到独立 Kafka topic

【部署】

- K8s HPA：CPU > 70% 或 QPS > 8000 扩容
- 最小 3 副本（PDB）
- Pod 资源：requests 2C4G, limits 4C8G

【验证】

1. 单 Pod 4C8G 下 QPS ≥ 1 万，P99 < 50ms
2. 鉴权失败率 < 0.1%
3. Kafka 发送成功率 ≥ 99.99%
4. 全链路 traceId 透传

【交付物】

- 完整代码 + WebFlux 配置
- Dockerfile + Helm Chart
- 压测脚本（wrk / JMeter）+ 压测报告
- README.md（含部署 / 监控 / 故障排查）
- Prometheus 告警规则

【禁止】

- 同步写存储
- 复杂业务逻辑
- Servlet 阻塞 IO
- 任何 System.out

【安全】

- 所有日志不输出 AppSecret
- IP/UA 信息脱敏存储
- 错误响应不透传内部信息

现在请生成完整服务代码。
```
