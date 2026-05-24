# 数据流架构

## 一、端到端数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│  采集层                                                          │
│  业务系统 ─SDK─► Collector (无状态网关) ─► Kafka [events_raw]   │
│  业务DB  ─CDC─► Debezium/Canal ─────────► Kafka [biz_cdc]      │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│  实时处理（Flink）         │    │  离线落盘                  │
│  · 异常检测                │    │  · ClickHouse dwd_event   │
│  · 实时聚合（DAU/PV）      │    │  · 分区按月，副本数 2     │
│  · 实时标签更新             │    │  · TTL 24 个月            │
│  · 实时反作弊               │    │                          │
└──────────┬───────────────┘    └────────────┬─────────────┘
           │                                  │
           ▼                                  ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│  实时存储                  │    │  离线计算（Spark/SQL）     │
│  Redis（实时指标）         │    │  · 每日 dws 聚合          │
│  ClickHouse（实时表）      │    │  · 标签生产                │
│                            │    │  · 物化视图刷新            │
└──────────┬───────────────┘    └────────────┬─────────────┘
           │                                  │
           └─────────────┬────────────────────┘
                         ▼
              ┌──────────────────────┐
              │  指标平台 + 标签平台   │
              │  （统一查询入口）       │
              └─────────┬────────────┘
                        ▼
              ┌──────────────────────┐
              │  应用层（看板/API/    │
              │  分群/触达/助理/洞察） │
              └──────────────────────┘
```

## 二、采集层详细

### Java SDK 上报流程

```
业务代码调用 client.track(...) 
   ↓
入内存阻塞队列（默认 100k 容量）
   ↓
后台消费线程批量取（默认 200/批）
   ↓
HTTP POST 到 Collector（带 HMAC 签名）
   ↓
成功：清队列
失败：本地磁盘队列暂存（Chronicle Queue）
   ↓
JVM 关闭：ShutdownHook flush
```

### Vue3 SDK 上报流程

```
v-track 触发 / 自动埋点
   ↓
batch 队列（5 条或 2s 触发）
   ↓
优先 navigator.sendBeacon
降级 fetch(keepalive)
最后 XHR
   ↓
失败：IndexedDB 缓存
下次启动重试
```

### Collector 处理流程

```
收到批量事件
   ↓
鉴权：AppKey + HMAC 签名校验（缓存命中 > 99%）
   ↓
校验：必填字段、事件元数据检查（Caffeine + Redis + Meta 服务）
   ↓
丰富：server_time、IP、geo、ua 解析、event_id
   ↓
异步写 Kafka（events_raw topic，按 app_id 分区）
   ↓
返回 ACK
```

## 三、数据分层

```
ODS (Raw)              DWD (Detail)            DWS (Summary)         ADS (Application)
─────────              ─────────────            ──────────────         ────────────────
events_raw       →     dwd_event         →     dws_user_daily   →     ads_funnel
biz_cdc          →     dwd_transaction   →     dws_org_monthly  →     ads_retention
                                               dws_feature_health     ads_health_score
                                               dws_segment_snapshot   ads_dashboard
```

| 层级 | 用途 | 存储 |
|---|---|---|
| ODS | 原始数据 | Kafka 短保留 |
| DWD | 明细事件 | ClickHouse ReplicatedMergeTree |
| DWS | 主题聚合 | ClickHouse + MaterializedView |
| ADS | 应用层数据 | ClickHouse + Redis 缓存 |

## 四、关键数据流场景

### 场景 1：实时指标查询

```
用户打开看板
   ↓
analytics-report 调 analytics-metric
   ↓
metric 查 Redis 缓存
   ├─ 命中 → 返回（< 50ms）
   └─ 未命中 → 查物化视图
      ├─ 命中 → 返回 + 写缓存（< 500ms）
      └─ 未命中 → 查 ClickHouse 明细 + 写物化 + 写缓存（< 3s）
```

### 场景 2：分群人数实时预估

```
用户拖拽配置分群条件
   ↓
analytics-segment 接收 DSL
   ↓
转换为 ClickHouse SQL（采样 10% 加速）
   ↓
返回预估人数（< 2s）
   ↓
确认创建：跑全量 + 落 segment_snapshot 表
```

### 场景 3：实时触达 Journey

```
用户触发事件（如付费成功）
   ↓
Kafka events_raw
   ↓
Flink 消费 + 匹配 Journey 触发条件
   ↓
触发 → 推送到 analytics-reach
   ↓
reach 检查疲劳度、退订状态
   ↓
调用通道适配器（站内信/邮件/短信/Push/企微）
   ↓
回写发送记录到 ClickHouse
   ↓
后续节点等待 / 流转
```

### 场景 4：决策助理对话

```
用户提问 "为什么 SaaS 续费下降"
   ↓
analytics-assistant 接收
   ↓
NLU 意图识别：归因类问题
   ↓
RAG 检索（元数据 + 决策追溯库 + 历史对话）
   ↓
LLM Plan：调用 Function 序列
   ├─ get_metric(NRR, last_week)
   ├─ get_anomaly_analysis(NRR)
   ├─ get_related_decisions("NRR下降")
   └─ generate_recommendation()
   ↓
LLM 整合结果，输出"数据 + 解释 + 建议"
   ↓
前端渲染卡片 + 操作按钮
   ↓
用户操作 → 触发后续 Function（如圈人、派单）
```

### 场景 5：智能洞察推送

```
每 5 分钟（实时指标）/ 每日（离线指标）
   ↓
analytics-insight 调度异动检测
   ↓
对每个核心指标跑算法（阈值/STL/Prophet）
   ↓
发现异动 → 维度归因（Adtributor）
   ↓
LLM 整合归因 + 检索相似历史决策
   ↓
生成"异动 + 归因 + 建议"
   ↓
按重要度推送（站内/企微/邮件）
   ↓
用户操作（处理/忽略/反馈）
   ↓
反馈写入学习层，调整下次推荐
```

## 五、数据质量保障

| 层级 | 检查 |
|---|---|
| 采集 | 必填字段、类型校验、事件注册检查、时间漂移 |
| Kafka | 消息积压、分区均衡 |
| Flink | Checkpoint 成功率、延迟 |
| ClickHouse | 副本同步、Merge 健康 |
| 业务规则 | 同环比异常、关联指标一致性 |

每项异常自动告警到数据 PM。

## 六、数据 SLA

| 数据 | 新鲜度 SLA | 备注 |
|---|---|---|
| 实时指标 | ≤ 1 分钟 | 关键业务 |
| T+0 准实时 | ≤ 1 小时 | 大部分指标 |
| T+1 日报 | ≤ 06:00 当日 | 离线聚合 |
| Session Replay | ≤ 24 小时 | 体积大 |
| 决策快照 | 实时 | 决策登记时冻结 |
