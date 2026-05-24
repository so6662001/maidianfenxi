# 提示词 07 · 分群引擎服务

> 前置：`00-master-prompt.md`。

---

## 提示词正文

```
你是数据中台工程师。请开发 analytics-segment 服务。

【目标】
拖拽式自助分群：实时预估 + 离线快照 + 跨产品圈人 + 与触达/AB 联动。

【技术】

- Spring Boot 3.2
- MyBatis-Plus + MySQL
- ClickHouse（行为查询）
- Redis + Redisson
- Kafka（分群快照变更事件）

【核心功能】

1. 分群规则配置（DSL）
2. 实时人数预估（采样 10% 加速）
3. 每日全量快照
4. 分群导出（CSV / Kafka / API）
5. 跨产品联合分群（基于 OneID）
6. 分群血缘追溯

【分群 DSL】

```json
{
  "name": "5月-高潜流失用户",
  "entityType": "user",
  "operator": "and",
  "conditions": [
    {
      "type": "event",
      "eventCode": "saas_invoice_export",
      "op": "count",
      "compareOp": ">=",
      "value": 5,
      "dateRange": "last_30_days"
    },
    {
      "type": "attribute",
      "field": "industry",
      "op": "in",
      "value": ["制造业", "电商"]
    },
    {
      "type": "tag",
      "tagCode": "high_churn_risk",
      "op": "=",
      "value": true
    }
  ]
}
```

【DSL → ClickHouse SQL】

实现 DSL 解析 + SQL 构建：
- event 条件 → JOIN dwd_event
- attribute 条件 → JOIN dim_user_daily
- tag 条件 → JOIN ads_user_tag

支持嵌套（and/or）、否定（not）。

【实时预估】

- 采样 10%（ClickHouse SAMPLE）加速
- 预估时长 < 2s
- 显示采样误差范围

【快照】

- 每日凌晨跑全量
- 存 t_seg_snapshot
- snapshot_data 大于 100k 用户 → 存 OSS（URL 指向）

【导出】

- CSV：异步任务 + 下载链接（OSS）
- Kafka：实时推送到指定 Topic（用于触达中心消费）
- API：批量拉取（分页）

【跨产品分群】

利用 OneID 联合查询：

```sql
SELECT distinct one_id
FROM analytics_oneid_mapping om
WHERE om.id_type = 'user'
  AND om.id_value IN (
    SELECT user_id FROM ... WHERE 条件A on saas
  )
  AND om.one_id IN (
    SELECT one_id FROM ... WHERE 条件B on ai
  )
```

【API】

```
POST /api/segment                   # 创建分群
PUT  /api/segment/{id}              # 更新
DELETE /api/segment/{id}            # 删除
GET  /api/segment/list              # 列表（分页）
GET  /api/segment/{id}              # 详情
POST /api/segment/preview           # 预估人数（无需保存）
GET  /api/segment/{id}/users        # 用户列表（分页）
POST /api/segment/{id}/export       # 异步导出（CSV / Kafka）
GET  /api/segment/{id}/snapshots    # 历史快照
GET  /api/segment/{id}/lineage      # 血缘
```

【性能】

- 预估 P95 ≤ 2s
- 快照执行 ≤ 30min（千万级用户）
- 导出 100w 用户 ≤ 10min

【验证】

1. DSL 解析正确性
2. 权限隔离（不可访问其他租户分群）
3. 大分群（1000w 用户）快照不 OOM
4. 跨产品分群准确性

【交付物】

- 完整代码 + 单测
- DSL 规范文档
- 接入示例
- Prometheus 指标

【安全】

- 行级权限注入
- 分群结果不允许跨租户
- 导出操作审计
- 敏感字段脱敏

现在请生成。
```
