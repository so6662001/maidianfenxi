# 用户分析中台 · 完整设计文档索引

> 本目录包含中台从战略 → 架构 → 设计 → 落地的全部设计文档，可直接作为研发、设计、运营团队的执行依据。

---

## 📚 文档结构

### 00 · 总览与愿景
- [01 - 中台愿景与定位](./00-overview/01-vision.md)
- [02 - 决策操作系统范式](./00-overview/02-paradigm.md)
- [03 - 产品路线图](./00-overview/03-roadmap.md)
- [04 - 新人入职手册](./00-overview/04-onboarding.md)

### 01 · 需求规约
- [01 - 业务背景与目标](./01-requirements/01-business-context.md)
- [02 - 用户角色与场景](./01-requirements/02-user-roles.md)
- [03 - 功能需求清单](./01-requirements/03-functional-requirements.md)
- [04 - 非功能性需求](./01-requirements/04-non-functional-requirements.md)

### 02 · 技术架构
- [01 - 系统总体架构](./02-architecture/01-system-architecture.md)
- [02 - 技术栈选型](./02-architecture/02-tech-stack.md)
- [03 - 微服务模块划分](./02-architecture/03-service-modules.md)
- [04 - 数据流架构](./02-architecture/04-data-flow.md)
- [05 - 对外集成模式](./02-architecture/05-integration-patterns.md)
- [06 - 术语表与词汇表](./02-architecture/06-glossary.md)
- [07 - 错误码字典](./02-architecture/07-error-codes.md)

### 03 · 业务模型与数据
- [01 - SaaS LCM 模型](./03-data-models/01-saas-lcm-model.md)
- [02 - AI 单位经济模型](./03-data-models/02-ai-unit-economics-model.md)
- [03 - 平台双边流动性模型](./03-data-models/03-platform-liquidity-model.md)
- [04 - 指标字典（60 指标）](./03-data-models/04-metrics-dictionary.md)
- [05 - 指标 SQL/DSL 实现](./03-data-models/05-metrics-sql.md)
- [06 - 数据 Schema 设计](./03-data-models/06-data-schema.md)
- [07 - 埋点规范与标准事件清单](./03-data-models/07-event-tracking-spec.md)
- [08 - 数据治理详细方案](./03-data-models/08-data-governance.md)

### 04 · 看板设计
- [01 - S-1 产品组合矩阵](./04-dashboards/01-s1-product-portfolio.md)
- [02 - S-2 战略健康度雷达](./04-dashboards/02-s2-strategic-health.md)
- [03 - S-3 PM 效能榜](./04-dashboards/03-s3-pm-performance.md)
- [04 - S-4 战略假设追踪](./04-dashboards/04-s4-strategic-hypothesis.md)
- [05 - S-5 决策待办池](./04-dashboards/05-s5-decision-pool.md)
- [06 - B-1 SaaS LCM 看板](./04-dashboards/06-b1-saas-dashboard.md)
- [07 - B-2 AI 单位经济看板](./04-dashboards/07-b2-ai-dashboard.md)
- [08 - B-3 平台双边流动性看板](./04-dashboards/08-b3-platform-dashboard.md)
- [09 - G-1 用户增长决策中枢](./04-dashboards/09-g1-growth-hub.md)

### 05 · 设计系统
- [01 - 设计 Tokens（色彩/字体/间距）](./05-design-system/01-design-tokens.md)
- [02 - 组件库映射](./05-design-system/02-component-library.md)
- [03 - 交互模式与状态规范](./05-design-system/03-interaction-patterns.md)

### 06 · Playbook 与触达
- [01 - Playbook 总览（37 个）](./06-playbooks/01-playbook-overview.md)
- [02 - SaaS Playbook（15 个）](./06-playbooks/02-saas-playbooks.md)
- [03 - AI Playbook（12 个）](./06-playbooks/03-ai-playbooks.md)
- [04 - 平台 Playbook（10 个）](./06-playbooks/04-platform-playbooks.md)
- [05 - 实验设计文档（37 个）](./06-playbooks/05-experiment-designs.md)
- [06 - 触达文案库](./06-playbooks/06-copy-templates.md)

### 07 · AI 助理与洞察引擎
- [01 - 助理产品架构](./07-ai-assistant/01-assistant-architecture.md)
- [02 - 30 个高频对话场景](./07-ai-assistant/02-conversation-scenarios.md)
- [03 - 智能洞察引擎](./07-ai-assistant/03-intelligent-insight-engine.md)
- [04 - AI 模型治理](./07-ai-assistant/04-model-governance.md)

### 08 · 决策追溯库
- [01 - 决策追溯库概览](./08-decision-traceability/01-decision-library-overview.md)
- [02 - 5 种决策模板](./08-decision-traceability/02-decision-templates.md)
- [03 - 自动复盘流程](./08-decision-traceability/03-review-workflow.md)
- [04 - 与三模型打通方案](./08-decision-traceability/04-integration-with-models.md)

### 09 · 用户增长
- [01 - 增长循环与分层](./09-user-growth/01-growth-loops-segmentation.md)
- [02 - 增长决策中枢](./09-user-growth/02-growth-decision-hub.md)

### 10 · 跨产品分析
- [01 - 跨产品分析 10 场景](./10-cross-product/01-cross-product-analysis.md)

### 11 · 嵌入式能力组件
- [01 - SDK 设计（Java + Vue3）](./11-embedded-components/01-sdk-design.md)
- [02 - Vue3 组件库清单](./11-embedded-components/02-component-library.md)
- [03 - 接入指南](./11-embedded-components/03-integration-guide.md)

### 12 · 对外 API
- [01 - OpenAPI 规范](./12-api-spec/01-openapi-spec.md)
- [02 - 鉴权与安全](./12-api-spec/02-authentication.md)

### 13 · OKR 与治理
- [01 - 中台 OKR 与成熟度模型](./13-okr-governance/01-middle-platform-okr.md)
- [02 - 变更管理流程](./13-okr-governance/02-change-management.md)
- [03 - 成本管理](./13-okr-governance/03-cost-management.md)

### 14 · 运维（生产部署与日常运维）
- [01 - 部署指南](./14-operations/01-deployment-guide.md)
- [02 - 监控与告警](./14-operations/02-monitoring-alerting.md)
- [03 - 故障排查 Runbook](./14-operations/03-troubleshooting.md)
- [04 - SLO/SLI/Error Budget](./14-operations/04-slo-sli.md)
- [05 - 灾备方案（DR/BCP）](./14-operations/05-disaster-recovery.md)

### 15 · 质量保证
- [01 - 测试策略总览](./15-quality-assurance/01-testing-strategy.md)
- [02 - 性能优化指南](./15-quality-assurance/02-performance-optimization.md)
- [03 - 混沌工程方案](./15-quality-assurance/03-chaos-engineering.md)

### 🤖 AI 开发提示词
- [开发提示词导航与使用指南](./prompts/README.md)
- 20 份子模块开发提示词位于 [`prompts/`](./prompts/) 目录

### 📋 项目协作
- [CONTRIBUTING.md](../CONTRIBUTING.md) · 贡献指南

---

## 🚀 推荐阅读顺序

| 角色 | 推荐顺序 |
|---|---|
| **总产品经理 / CEO** | 00 → 01 → 13 → 04 |
| **PM** | 01 → 04 → 06 → 09 |
| **设计师** | 04 → 05 |
| **后端研发** | 00-onboarding → 02 → 03 → 11 → 12 → 14-15 → prompts/ |
| **前端研发** | 00-onboarding → 02 → 04 → 05 → 11 → 15 → prompts/ |
| **数据 PM** | 03 → 04 → 06 → 08-decision |
| **运营** | 06 → 07 → 09 |
| **AI 工程师** | 07 → 08 → 15 → prompts/11-ai-assistant.md |
| **SRE / DevOps** | 02 → 14 → 15 → prompts/18-19 |
| **新员工** | 00-onboarding 入门 → 按角色 |

## ✅ 落地执行流程

```
1. 选定 MVP 范围（见 00-overview/03-roadmap.md）
2. 团队成立 & 角色分工
3. 设计师按 04 / 05 出 Figma 高保真
4. 数据 PM 按 03 落指标平台
5. 后端按 02 / 03 / 11 / 12 + prompts/ 启动开发
6. 前端按 02 / 04 / 05 / 11 + prompts/ 启动开发
7. 运营按 06 / 09 准备 Playbook 上线
8. MVP 跑通 → V1 扩展 → V2 全量
```

## 📞 维护

本文档为活文档，落地过程中遇到的细化、调整、决策都应回写到对应文档。
