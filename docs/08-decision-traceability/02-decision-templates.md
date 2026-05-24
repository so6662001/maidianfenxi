# 5 种决策模板（详细）

## 通用字段（所有模板）

```yaml
基础信息（30 秒填）:
  decision_id: 自动生成 D-YYYY-MMDD-NNN
  title: 文本 ≤ 50 字  必填
  type: 选择 (产品/资源/战略/组织/应急)  必填
  importance: P0/P1/P2  必填
  decided_by: 当前用户（自动）
  decided_at: 当前时间（自动，可改为补登）
  collaborators: 多选用户  可选
  related_orgs: 多选企业  可选
  related_pms: 多选 PM  可选
  visibility: 公司/部门/仅决策人  必填

决策内容（2-3 分钟，AI 帮填一半）:
  summary: 富文本  AI 起草
  background: 富文本  AI 起草（抓取相关指标/异动）
  options: 结构化（至少 2 个）  AI 推荐替代方案
  chosen_option: A/B/C  必填
  rationale: 富文本  必填
  expected_impact: 结构化 30/60/90/180  必填 + AI 推荐 KPI

复盘配置（30 秒）:
  review_dates: 默认 30/60/90/180
  reviewer: 默认决策人
  tracking_metrics: 自动从 expected_impact 带入
  alert_threshold: 自动（偏离 ≥ 30% 提醒）

附件:
  snapshot: 数据快照（自动）
  attachments: 上传文件
  links: 关联链接
```

---

## 模板 1 · 产品决策

```yaml
type: 产品决策
importance: P1（默认）
预填:
  related_product: <自动从看板带入>
  related_hypothesis: <可选>

特有字段:
  影响范围:
    user_count: <自动从数据>
    affected_features: <选>
    affected_departments: <选>

预填示例:
  title: 砍掉"高级筛选"功能
  background: |
    · 该功能使用率仅 12%
    · 维护成本 30 人月/年
    · 客服工单 5%
    · 健康度评分 28（瘦狗象限）
  options:
    A. 立即砍掉
    B. 灰度执行
    C. 维持现状
  expected_impact:
    30d: { user_complaints: 减少 < 20%, dev_capacity: +30人月 }
    90d: { 替代功能采纳率: ≥ 50% }
    180d: { 总满意度: 不下降 }

复盘要点:
  - 用户反馈如何
  - 数据指标达成情况
  - 是否引发意外问题
  - 是否需要回滚
```

---

## 模板 2 · 资源决策

```yaml
type: 资源决策
importance: P0/P1
预填:
  related_products: <多选>

特有字段:
  resource_type: 预算 / 人力 / 流量 / 算力 / 其他
  resource_scale:
    before: <数字>
    after: <数字>
    delta: <自动>
  source_destination:
    from: <产品/团队>
    to: <产品/团队>
  roi_estimation:
    expected_roi: <数字>
    historical_similar: <自动调取案例>
  opportunity_cost:
    not_doing_consequence: <文本>
    sacrifice: <文本>
  expected_impact:
    quarterly: 资源投入产出
    yearly: 战略指标
  risks:
    team_change: <文本>
    feasibility: <文本>
    exit_cost: <文本>

复盘要点:
  - ROI 实际 vs 预期
  - 是否影响其他项目
  - 资源利用率
```

---

## 模板 3 · 战略决策

```yaml
type: 战略决策
importance: P0（默认）
影响时长: 12-36 个月
预填:
  related_hypothesis: 强制 ≥ 1 个

特有字段:
  strategic_background:
    industry_trend: <文本>
    competitor_actions: <文本>
    company_position: <文本>
  core_hypotheses: ≥ 3 条
    - "我们认为 X 会发生，因为 Y"
  long_term_vision: 3 年期望状态
  milestones:
    Q1: 验证 {子假设}
    Q2: ...
    Q3: ...
    Q4: ...
  key_resources:
    人力: <数>
    预算: <数>
    时间: <数>
  exit_conditions: |
    "如果 {失败信号} 出现，则 {退出动作}"
  decision_gates:
    Q1: 是否继续
    Q2: 是否调整
    Q4: 是否扩大

预填示例:
  title: 投资 AI 业务成为第二增长曲线
  core_hypotheses:
    - 行业 AI 渗透率 5% → 30% (24 个月)
    - 我们的 SaaS 数据资产能形成 AI 差异化
    - 客户愿为 AI 多付 30% 溢价
  key_resources: 60 人 + 5000 万 + 24 个月
  exit_conditions:
    - 12 月未达 ARR 5000 万
    - 单位毛利持续 < 0 超过 6 月

复盘节点: 90 / 180 / 365 天
复盘要点:
  - 假设是否被验证
  - 是否达到阶段里程碑
  - 是否需要 Pivot
  - 学到了什么
```

---

## 模板 4 · 组织决策

```yaml
type: 组织决策
importance: P0/P1
保密级别: 部门内 / 高管 / 仅决策人

特有字段:
  org_change_type: 任命 / 调岗 / 招聘 / 裁撤 / 架构调整
  affected_persons: <人员列表，敏感>
  org_before: <组织树截图或描述>
  org_after: <组织树>
  decision_basis:
    business_need: <文本>
    capability_match: <数据 S-3 PM 效能榜>
    performance_records: <数据>
    strategic_need: <文本>
  communication_plan:
    announcement_time: <日期>
    sequence: 先 1v1 后公告
    draft_message: <文本>

复盘节点: 30 / 90 / 180 天
复盘要点:
  - 团队反应
  - 业务交接是否顺畅
  - 当事人状态
  - 业务影响
```

---

## 模板 5 · 应急决策

```yaml
type: 应急决策
importance: P0（自动）
登记时效: 决策后 24h 内必须

精简登记（24h 内）:
  title: 一句话描述
  crisis_type: 系统故障 / 数据安全 / 公关 / 客户投诉 / 法务 / 其他
  decided_at: <时间> + decided_by
  immediate_action: 一段话描述做了什么
  severity: P0

补充登记（24-72h）:
  full_timeline: <时间线>
  impact_scope: <影响范围>
  direct_cause: <直接原因>
  root_cause: 5 Why 分析
  improvement_actions: <改进>
  prevention_mechanism: <预防机制>

复盘节点: 7 / 30 / 90 天
复盘要点:
  - 决策是否正确
  - 反应是否及时
  - 是否避免类似问题
  - 流程是否需要改进

特殊机制:
  - 24h 必须登记，否则上级被通知
  - 复盘对事不对人，鼓励"无指责复盘"
  - 复盘结论自动加入"应急响应手册"
```

### 应急决策示例

```yaml
title: 合同模块上传服务故障 2h
crisis_type: 系统故障
immediate_action: 立即回滚 v2.5.1 + 启动备用方案 + 客户公告致歉
severity: P0 - 影响 1000+ 企业用户

24h 后补充:
  full_timeline:
    09:13 监控告警
    09:18 确认故障
    09:25 决策回滚
    09:45 回滚完成
    10:00 服务恢复
    10:30 客户公告
  direct_cause: v2.5.1 引入的文件校验逻辑 bug
  root_cause:
    1. 为什么有 bug？校验逻辑边界条件考虑不全
    2. 为什么没测出？测试用例覆盖不足
    3. 为什么测试不足？发布流程跳过了集成测试
    4. 为什么跳过？赶上线进度
    5. 为什么赶进度？需求评估时机激进
  improvement_actions:
    - 文件类操作强制集成测试
    - 灰度发布机制
    - 监控加上"上传成功率"维度
```

---

## 通用机制

### 智能模板推荐

```
用户点击"+ 新建决策"
   ↓
助理弹窗："是什么类型的决策？"
   ↓
用户描述（自然语言）
   ↓
助理自动判断类型 + 推荐模板
   ↓
预填字段（基于当前上下文）
   ↓
用户补充 + 提交
```

### 字段自动填充

| 字段 | 来源 |
|---|---|
| 决策时间 | 当前时间 |
| 决策人 | 当前登录用户 |
| 关联产品 | 当前看板 |
| 数据快照 | 当前看板截图 + SQL + 数据集 |
| 业务背景 | 抓取相关指标 + 异动 |
| 历史相似案例 | 决策追溯库语义搜索 |
| 预期效果 KPI | 基于决策类型推荐 |
