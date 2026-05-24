# 提示词 08 · Journey 与触达服务

> 前置：`00-master-prompt.md`。参考：`docs/06-playbooks/`。

---

## 提示词正文

```
你是营销自动化工程师。请开发 analytics-reach 服务。

【目标】
Journey 可视化编排 + 多通道触达 + 疲劳度控制 + 效果追踪。

【技术】

- Spring Boot 3.2
- MyBatis-Plus + MySQL
- Flink CEP（触发条件评估）
- Kafka（Journey 触发事件 / 任务调度）
- Redisson（分布式锁 + 疲劳度计数）
- 各通道适配器

【核心功能】

1. Journey 流程编排（拖拽节点）
2. 触发条件（事件 / 标签变化 / 定时）
3. 多分支与等待（If/Else、A/B、定时）
4. 多通道触达（站内 / 邮件 / 短信 / Push / 企微 / 钉钉）
5. 任务调度与重试
6. 疲劳度控制
7. 退订管理
8. 效果追踪

【Journey 数据结构】

```json
{
  "id": "j_001",
  "name": "高潜流失挽留",
  "trigger": {
    "type": "tag_change",
    "config": { "tag": "high_churn_risk", "newValue": true }
  },
  "nodes": [
    {
      "id": "n1",
      "type": "send_inapp",
      "templateId": 100,
      "next": "n2"
    },
    {
      "id": "n2",
      "type": "wait",
      "duration": "2d",
      "next": "n3"
    },
    {
      "id": "n3",
      "type": "send_email",
      "templateId": 101,
      "abTest": {
        "control": 50,
        "treatment": 50
      },
      "next": "n4"
    },
    {
      "id": "n4",
      "type": "condition",
      "if": { "metric": "open_email_recent_3d", "op": "=", "value": true },
      "then": "n5",
      "else": "n6"
    },
    ...
  ]
}
```

【触达通道适配器】

抽象接口：

```java
public interface ChannelAdapter {
    String getChannelType();
    void send(ReachTask task) throws ChannelException;
    boolean supportsBatch();
}
```

实现：
- InAppChannelAdapter（站内消息，WebSocket 推送）
- EmailChannelAdapter（SMTP / 阿里云邮件 / SendGrid）
- SmsChannelAdapter（阿里云短信 / 腾讯云短信）
- PushChannelAdapter（极光 / 个推 / Firebase）
- WecomChannelAdapter（企微应用消息）
- DingTalkChannelAdapter（钉钉机器人）

【疲劳度控制】

Redis 计数：
- `analytics:fatigue:user:{userId}:24h` → 当日累计触达数（TTL 24h）
- 单用户 24h 累计 ≤ 3 次
- 同 Journey 30 天内不重复触达同一用户

【退订管理】

- 退订表 t_reach_unsubscribe
- 邮件/短信必须含退订链接
- 站内/企微提供"减少推送"开关
- 营销/系统/产品 三类分别管理

【触发器】

1. **事件触发**：Flink CEP 实时匹配
2. **标签变化触发**：监听 Kafka tag_change 事件
3. **定时触发**：XXL-Job + cron 表达式

【任务调度】

```
触发器 → Kafka journey_trigger
   ↓
Journey Engine 消费
   ↓
为每个用户创建 JourneyRun（进入第一个节点）
   ↓
节点执行（按类型）
   ├─ send_xxx → 创建 ReachTask → 通道发送
   ├─ wait → 调度延迟任务
   ├─ condition → 评估 → 跳转
   └─ ab_test → 哈希分流
   ↓
节点完成 → 更新 JourneyRun.current_node → 进入下一节点
```

【API】

```
POST /api/journey                       # 创建
PUT  /api/journey/{id}                  # 更新
PUT  /api/journey/{id}/status           # 启用/暂停
GET  /api/journey/list
GET  /api/journey/{id}/runs             # 用户进入记录
GET  /api/journey/{id}/stats            # 效果统计
POST /api/reach/send                    # 即时触达（不走 Journey）
POST /api/reach/unsubscribe             # 退订
```

【效果追踪】

- 每个节点：进入数 / 完成数 / 流失数
- 每个 ReachTask：发送 / 到达 / 打开 / 点击 / 转化 / 失败
- 整体 Journey：起点 → 终点转化率 + ROI
- AB 实验结果归档到决策追溯库

【性能】

- 单 Journey 支持 100w 用户并发
- 触达任务调度延迟 < 5s
- 通道发送成功率 ≥ 99%

【验证】

1. Journey 完整流转（含分支 / 等待 / AB）
2. 疲劳度生效
3. 退订生效
4. 通道故障降级
5. 大批量发送不阻塞

【交付物】

- 完整服务代码
- Journey 编辑器 API
- 通道适配器（≥ 6 个）
- 效果统计 API
- 监控指标
- 测试

【安全】

- 触达前必查退订
- 邮件/短信合规审核
- 防刷（同一用户 1 小时内同模板限发）
- 内容审核（敏感词过滤）
- 操作审计

现在请生成。
```
