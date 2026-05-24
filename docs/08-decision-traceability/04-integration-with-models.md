# 决策追溯库 · 与三模型打通方案

## 一、打通总体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    三套业务模型看板                          │
│   SaaS LCM        AI 单位经济        平台双边流动性          │
└─────────┬───────────────┬──────────────────┬───────────────┘
          │               │                  │
          ▼               ▼                  ▼
   ┌──────────────────────────────────────────────┐
   │           智能洞察引擎（异动+归因+建议）       │
   └─────────────────────┬────────────────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  决策点触发  │ ← 异动 / 评审 / Playbook 启动 / 人工
                  └──────┬──────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   决策追溯库登记      │
              │  · 数据快照自动冻结  │
              │  · 模型上下文写入    │
              │  · 关联 Playbook    │
              │  · 关联战略假设      │
              │  · 关联 PM 效能榜    │
              └──────────┬──────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  自动复盘    │ ← 调取模型当时 vs 现在
                  └──────┬──────┘
                         │
                         ▼
              ┌─────────────────────┐
              │    知识沉淀库        │
              │  → 反哺 AI 建议      │
              │  → 反哺 Playbook 优化 │
              │  → 反哺指标平台      │
              └─────────────────────┘
```

## 二、数据 Schema 与表设计

详见 [数据 Schema](../03-data-models/06-data-schema.md) 中 `analytics_decision` 库部分。

核心表：
- `t_decision` 决策主表
- `t_decision_review` 复盘记录
- `t_decision_snapshot` 数据快照
- `t_strategic_hypothesis` 战略假设

## 三、模型联动详细方案

### 与 SaaS LCM 联动

```
SaaS 看板检测到 NRR 异动
   ↓
智能洞察归因 → 流失加剧
   ↓
助理建议启动 P-SE-001
   ↓
用户点击 → 决策追溯库登记
   预填：
     type: 产品/资源决策
     related_model: saas
     related_metrics: [M-SE-001 NRR]
     related_playbook: P-SE-001
     snapshot: SaaS LCM 看板快照
   ↓
30/60/90 天后自动复盘
   调取：NRR 变化、续约率、流失原因
```

### 与 AI 单位经济联动

```
AI 看板检测到单次毛利下降
   ↓
助理建议启动 P-AU-001 模型路由优化
   ↓
登记决策：
   type: 资源决策
   related_model: ai
   related_metrics: [M-AU-001, M-AE-001]
   related_playbook: P-AU-001
   snapshot: AI 单位经济看板
   特殊：实验 EXP-007 自动关联
   ↓
14 天后复盘
   调取：成本、采纳率、点踩率
```

### 与平台双边流动性联动

```
平台看板检测到类目失衡
   ↓
助理建议启动 P-B-001
   ↓
登记决策：
   type: 资源决策
   related_model: platform
   related_metrics: [M-B-001]
   related_playbook: P-B-001
   snapshot: 供需热力地图 + 失衡 Top10
   ↓
30/60 天复盘
   调取：该类目供需比恢复、撮合率
```

## 四、跨模型决策（跨产品）

某些决策涉及多产品（如资源转移、跨产品定价、战略调整）：

```
决策类型：cross
related_model: cross
related_products: [saas, ai]
snapshot: 多模型聚合快照
复盘指标: 各模型相关指标 + 跨模型联动指标（如 LTV 跨产品对比）
```

## 五、与战略假设（S-4）联动

```
每个战略级决策必须关联至少 1 个战略假设
  ↓
战略假设进度根据相关决策自动更新
  ↓
假设偏离 → 触发新的决策评审
  ↓
形成"假设 → 决策 → 验证 → 调整"闭环
```

## 六、与 PM 效能榜（S-3）联动

```
每个决策的 decided_by → PM
  ↓
复盘结论（达成/未达成）→ PM 决策准确率
  ↓
按时间周期（季度）聚合 → 排名
  ↓
反哺 S-3 排名
```

## 七、知识沉淀反哺机制

详见 [复盘流程](./03-review-workflow.md) 第五节。

## 八、数据接口

### 决策登记
```
POST /api/decision
Body: {
  title, type, importance,
  related_model, related_metrics, related_playbook, related_hypothesis,
  background, options, chosen_option, rationale,
  expected_impact, review_dates,
  snapshot: { dashboard_id, data_url, sql_queries }
}
Response: { decision_id, code }
```

### 决策复盘
```
POST /api/decision/{id}/review
Body: {
  review_phase, actual_impact, deviation_analysis,
  conclusion, lessons_learned, knowledge_tags, shareable
}
```

### 检索相似决策
```
POST /api/decision/search
Body: { query, type?, model?, status?, top_k? }
Response: { decisions: [...with similarity scores] }
```

### 自动复盘初稿
```
POST /api/decision/{id}/auto-review-draft
Effect: 调取数据 + LLM 生成初稿 → 发给责任人
```

## 九、监控

### 决策追溯库自身指标
- 登记数 / 季度
- P0/P1 决策登记覆盖率（目标 ≥ 80%）
- 复盘完成率 / 准时率
- 决策达成率
- 案例库被检索次数
- 反哺到 Playbook 优化的数量
