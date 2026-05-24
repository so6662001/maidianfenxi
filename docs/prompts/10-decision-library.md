# 提示词 10 · 决策追溯库

> 前置：`00-master-prompt.md`。参考：`docs/08-decision-traceability/`。

---

## 提示词正文

```
你是中台应用工程师。请开发 analytics-decision 服务。

【目标】
组织决策记忆系统：登记 + 数据快照 + 自动复盘 + 知识沉淀 + 多模块联动。

【技术】

- Spring Boot 3.2
- MyBatis-Plus + MySQL
- Elasticsearch（决策全文检索 + 语义检索）
- LangChain4j（复盘初稿生成）
- OSS / MinIO（数据快照存储）
- XXL-Job（复盘提醒调度）

【核心功能】

1. 决策登记（5 模板：产品/资源/战略/组织/应急）
2. 数据快照自动冻结
3. 自动复盘（30/60/90/180 天）
4. 知识沉淀（标签 + 检索）
5. 与 PM 效能榜联动
6. 与战略假设联动
7. 与决策助理联动（提供相似案例）

【表结构】

```sql
t_decision                     # 决策主表
t_decision_review              # 复盘记录
t_decision_snapshot            # 数据快照
t_strategic_hypothesis         # 战略假设
```

详见 docs/03-data-models/06-data-schema.md。

【决策登记 API】

```
POST /api/decision
Body:
{
  title, type, importance,
  decided_by (自动), collaborators, decided_at (自动),
  related_model, related_metrics, related_playbook, related_hypothesis,
  related_orgs, related_pms,
  background, options (≥ 2), chosen_option, rationale,
  expected_impact: { "30d": [...], "60d": [...], "90d": [...], "180d": [...] },
  review_dates: ["30d", "60d", "90d", "180d"],
  reviewer, visibility,
  snapshot: { dashboard_id, params, time }
}
Response: { decision_id, code }
```

【数据快照机制】

```
决策提交时：
   ↓
1. 调取当前看板数据 → 保存原始 JSON 到 OSS
2. 调用看板服务截图 → 保存 PNG 到 OSS
3. 记录查询 SQL + 参数
4. 写 t_decision_snapshot
   ↓
快照不可删除（除非 GDPR 请求）
```

【5 种模板差异化】

每种模板预填不同字段，详见 docs/08-decision-traceability/02-decision-templates.md。

实现：
- 模板配置存配置中心（Nacos）
- 前端根据 type 渲染不同表单
- 后端校验时根据 type 加载不同 validator

【自动复盘流程】

XXL-Job 每日跑：

```
扫描 t_decision_review 中 review_date <= today 的待复盘
   ↓
对每个：
   1. 调取决策时数据快照
   2. 调取当前最新数据（同样的指标）
   3. 对比生成"实际 vs 预期"差异表
   4. LLM 撰写复盘初稿（结构化 + 自然语言）
   5. 推送给 reviewer（站内 + 邮件 + 待办）
   6. 创建 t_decision_review 草稿（待 reviewer 确认）
```

【LLM 复盘初稿 Prompt】

```
基于以下数据，撰写决策复盘初稿：

决策时刻数据快照：
{snapshot_at_decision}

当前数据：
{current_data}

决策预期：
{expected_impact}

请输出：
1. 进度评估（达成/部分达成/未达成）
2. 实际效果 vs 预期 对比表
3. 偏差分析（≤ 3 个可能原因）
4. 后续建议（继续/调整/终止）
```

【知识沉淀】

复盘提交后：
- 抽取关键词作为 knowledge_tags
- 向量化（Embedding）存 Elasticsearch
- 标记 shareable 进入"案例库"

【与决策助理联动】

```
POST /api/decision/search/similar
Body: { query, type?, model?, top_k? }
Response: 相似案例列表（带 similarity score）

助理在用户做新决策时自动调用，推荐参考案例
```

【与 PM 效能榜联动】

- 每个决策关联 decided_by
- 复盘结论 → PM 决策准确率
- 季度聚合 → S-3 排名

【与战略假设联动】

- 战略级决策必须关联 ≥ 1 个假设
- 假设进度根据相关决策自动更新
- 假设偏离时触发新决策评审

【强制登记机制】

- P0/P1 决策必须登记
- Playbook 启动时自动创建决策草稿
- 高管未登记 → 第 1 天提醒、第 3 天上级通知

【API 清单】

```
POST   /api/decision                        # 登记
GET    /api/decision/{id}
PUT    /api/decision/{id}                   # 修改（关键字段冻结后不可改）
DELETE /api/decision/{id}                   # 仅草稿可删
POST   /api/decision/{id}/review            # 提交复盘
GET    /api/decision/{id}/review/draft      # 获取自动复盘初稿
POST   /api/decision/search                 # 全文 + 语义检索
POST   /api/decision/search/similar         # 相似案例
GET    /api/decision/case-library           # 案例库浏览
POST   /api/hypothesis                      # 战略假设
GET    /api/pm/{userId}/decisions           # PM 个人决策列表
GET    /api/pm/{userId}/accuracy            # PM 决策准确率
```

【性能】

- 决策登记 < 1s
- 决策检索 < 500ms
- 相似案例检索 < 2s
- 复盘初稿生成 < 30s（异步）

【验证】

1. 5 种模板差异化预填
2. 数据快照不可篡改
3. 自动复盘提醒准确
4. LLM 初稿质量人工抽检 ≥ 80% 可用
5. 权限隔离（敏感决策仅决策人可见）

【交付物】

- 完整服务代码
- 5 种模板配置（Nacos）
- LLM Prompt 模板
- 复盘报告模板
- 测试

【安全】

- 决策修改全留痕
- 涉敏决策（组织变动）严控权限
- 数据快照永久不删（除 GDPR）
- 审计完整

现在请生成。
```
