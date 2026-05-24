# 故障排查 Runbook

> 每个常见故障的诊断 → 缓解 → 根因 → 修复 → 复盘 SOP。

## 通用排查流程

```
1. 确认告警 → 评估影响范围
2. 立即缓解（保用户）：回滚 / 限流 / 切流 / 重启
3. 收集证据：日志 / 监控 / 链路 / 配置
4. 根因分析（5 Why）
5. 修复 + 灰度验证
6. 完整复盘 → 决策追溯库
7. 改进措施（监控加固 / 演练 / 自动化）
```

## 常见故障 Runbook

### F-001 · 服务全量不可用

**告警**：`ServiceDown` / 所有副本 0

**排查**：
```bash
# 1. 查 Pod 状态
kubectl get pods -n analytics-prod -l app=analytics-data

# 2. 查事件
kubectl describe pod -n analytics-prod <pod-name>

# 3. 查日志
kubectl logs -n analytics-prod <pod-name> --tail=200
kubectl logs -n analytics-prod <pod-name> --previous  # 上次崩溃日志

# 4. 查节点
kubectl get nodes
kubectl describe node <node-name>
```

**常见原因**：
- OOM：调大 limits 或排查内存泄漏
- 配置错误：检查 Nacos 配置回滚
- 镜像拉不到：检查 Harbor / 镜像 tag
- 节点资源不足：扩容节点或迁移 Pod
- 上游依赖故障：检查 MySQL/Redis/Kafka

**缓解**：
```bash
# 立即回滚
kubectl argo rollouts undo analytics-data -n analytics-prod

# 扩容副本
kubectl scale deployment analytics-data --replicas=10 -n analytics-prod
```

### F-002 · 接口高错误率

**告警**：`HighErrorRate` > 5%

**排查**：
```
1. Grafana 错误率看板 → 哪个服务 / 接口 / 错误码
2. ELK 错误日志聚类 → 找共性
3. SkyWalking 链路 → 上下游问题
4. 最近变更：发布 / 配置 / 数据
```

**常见原因**：
- 新版本 Bug → 回滚
- 上游服务故障 → 修上游
- 数据库慢查询 → 限流 + 优化
- 配置错误 → 修配置
- 流量突增 → 限流 + 扩容

### F-003 · 接口延迟飙升

**告警**：`HighLatencyP95` > 5s

**排查**：
```
1. SkyWalking 链路追踪 → 找最慢环节
2. MySQL/ClickHouse 慢查询日志
3. Redis 命中率
4. JVM GC 情况
5. 上游 API 响应时间
```

**常见原因**：
- SQL 慢 → 加索引 / 优化 SQL
- 缓存失效 → 缓存预热
- LLM 响应慢 → 模型路由 / 超时配置
- GC 频繁 → 调 JVM 参数
- 网络延迟 → 检查网络

### F-004 · Kafka Lag 过高

**告警**：`KafkaLagHigh` > 100k

**排查**：
```bash
# 查 Consumer Lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group analytics-events

# 查 Topic 状态
kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic events_raw
```

**缓解**：
- 扩容消费者 Pod
- 临时调大消费者批次
- 检查下游瓶颈（ClickHouse 写入慢？）
- 实在不行：消费者跳过部分消息（损失数据）

### F-005 · ClickHouse 查询慢 / OOM

**告警**：`SlowQueryHigh` 或 `ClickHouse OOM`

**排查**：
```sql
-- 当前运行查询
SELECT query_id, query, user, elapsed FROM system.processes ORDER BY elapsed DESC LIMIT 10;

-- 历史慢查询
SELECT query, query_duration_ms, memory_usage FROM system.query_log 
WHERE event_date = today() AND query_duration_ms > 10000 ORDER BY query_duration_ms DESC LIMIT 20;
```

**缓解**：
```sql
-- 立即 Kill
KILL QUERY WHERE query_id = 'xxx';

-- 限制查询资源
SET max_memory_usage = 10000000000;
SET max_execution_time = 60;
```

**根因**：
- 缺少分区过滤 → 修 SQL
- JOIN 大表 → 加物化视图
- 全表扫描 → 加索引（Skip Index）
- 数据倾斜 → 重新分片

### F-006 · LLM 成本异常飙升

**告警**：`LLMCostSpike`

**排查**：
```
1. 查 assistant_llm_cost 看板 → 哪个模型 / 用户 / 场景
2. 查 assistant_chat 日志 → 看是否被刷
3. 查 Token 消耗：是否输入/输出异常长
4. 查模型路由命中率：是否大模型用太多
```

**缓解**：
- 临时限流（按用户 QPS 调低）
- 切换到更便宜的模型
- 启用更严格的输出长度限制
- 怀疑攻击：黑名单 IP / 用户

**根因**：
- 用户滥用 → 加配额
- 路由失效 → 修路由策略
- Prompt 设计差 → 优化系统 Prompt
- 攻击 → 防护加固

### F-007 · 数据库主从延迟

**告警**：`MySQLReplicationLag` > 10s

**排查**：
```sql
SHOW SLAVE STATUS;  -- Seconds_Behind_Master
```

**常见原因**：
- 主库写入过快
- 大事务
- 网络问题

**缓解**：
- 临时把读切回主库
- 大事务拆分
- 增加从库

### F-008 · Redis 内存满

**告警**：`RedisMemoryFull` > 90%

**排查**：
```
redis-cli --bigkeys
redis-cli info memory
```

**缓解**：
- 立即清理过期 Key
- 找大 Key 删除
- 扩容
- 调整 maxmemory-policy（allkeys-lru）

### F-009 · 数据丢失

**告警**：业务方反馈数据缺失

**排查**：
```
1. 查 Collector 日志：是否收到
2. 查 Kafka 日志：是否成功写入
3. 查 Flink 日志：是否消费失败
4. 查 ClickHouse 落盘：是否写入
5. 查 DLQ：是否被异常拒绝
```

**根因**：
- SDK 配置错误 → 业务方修正
- Collector 校验过严 → 调整规则
- Kafka 写入失败 → 检查 Broker
- Flink 任务失败 → 重启任务
- ClickHouse 写入失败 → 检查存储

**补救**：
- DLQ 数据重放
- 业务库 CDC 重跑
- 实在无法恢复：通知业务方 + 公告

### F-010 · 决策助理回答不准

**告警**：`AssistantThumbsdownHigh` 或用户反馈

**排查**：
```
1. 看具体对话日志（脱敏后）
2. 是模型问题还是 Prompt 问题？
3. 是 RAG 检索不准还是 LLM 错？
4. 是权限误判还是数据真的不存在？
```

**改进**：
- 优化系统 Prompt
- 调整 RAG 检索策略
- 切换模型
- 加入 few-shot 示例
- 改进 Function Calling Plan

### F-011 · 嵌入组件白屏

**业务方反馈**：组件不显示

**排查**：
```
1. 浏览器控制台错误
2. 网络请求是否成功
3. Token 是否有效
4. 跨域是否配置正确
5. 中台 API 是否可用
```

**常见原因**：
- Token 过期 → 业务方刷新 Token
- CORS 错误 → 检查 CORS 配置
- 网络阻断 → 检查 WAF / CDN
- 组件版本不兼容 → 降级或升级

### F-012 · 触达大规模失败

**告警**：`ReachDeliveryRateLow`

**排查**：
```
1. 哪个通道？（邮件/短信/Push/企微）
2. 第三方服务是否正常
3. 退订名单是否异常
4. 配额是否用尽
5. 内容是否被反垃圾系统拦截
```

**缓解**：
- 切换备用通道
- 降低发送速率
- 暂停大规模任务
- 联系第三方服务商

## 紧急联系人清单

| 角色 | 联系方式 | 责任 |
|---|---|---|
| L1 oncall | 企微：@analytics-oncall-l1 | 7×24 响应 |
| L2 oncall | 企微：@analytics-oncall-l2 | 工作时间 + 夜间紧急 |
| L3 oncall（架构） | 企微：@analytics-arch | 重大故障升级 |
| 中台负责人 | 企微：@analytics-lead | P0 升级 |
| 安全负责人 | 企微：@security-lead | 数据泄露 / 入侵 |
| DBA | 企微：@dba | 数据库相关 |
| 网络运维 | 企微：@network-ops | 网络故障 |
| 第三方供应商 | 见 vendor 联系表 | 第三方服务故障 |

## 应急通信群

- `analytics-emergency` 企微群：所有 oncall + 高管
- 故障期间在群内同步进展，每 30 分钟一次

## 故障复盘模板

每次 P0/P1 故障必须复盘，登记到决策追溯库（应急决策模板）：

```yaml
故障 ID: INC-2026-0524-001
等级: P0
开始时间: 2026-05-24 09:13
解决时间: 2026-05-24 10:00
持续时长: 47 min
影响范围: 全公司 SaaS 用户（约 2,000 家企业）
直接原因: 合同模块上传服务故障
根因（5 Why）:
  1. 为什么有 bug？校验逻辑边界条件考虑不全
  2. 为什么没测出？测试用例覆盖不足
  3. 为什么测试不足？发布流程跳过了集成测试
  4. 为什么跳过？赶上线进度
  5. 为什么赶进度？需求评估时机激进
时间线:
  09:13 监控告警
  09:18 确认故障
  09:25 决策回滚
  09:45 回滚完成
  10:00 服务恢复
  10:30 客户公告
改进措施:
  - 文件类操作强制集成测试
  - 灰度发布机制
  - 监控加上"上传成功率"维度
  - 流程改进：禁止跳过集成测试
经验沉淀:
  - 知识标签：[文件上传] [发布流程] [P0 故障]
  - 自动加入"应急响应手册"
```

## 演练计划

- **季度**：模拟 1 次 P0 故障演练（混沌工程）
- **月度**：oncall 应急响应演练
- **新员工**：入职 1 周内必须看完此 Runbook + 1 次模拟演练
