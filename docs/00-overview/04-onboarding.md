# 新人入职手册

> 中台团队任何新成员入职 7 天内必读，30 天内消化。

## 第 1 天：理解

### 必看
- [中台愿景与定位](./01-vision.md)
- [决策操作系统范式](./02-paradigm.md)
- [产品路线图](./03-roadmap.md)

### 任务
- [ ] 阅读上述 3 份文档
- [ ] 与 mentor 1v1 介绍业务背景
- [ ] 加入团队企微 / Slack 频道
- [ ] 申请必要权限（GitLab / Nacos / Grafana / 决策追溯库）

## 第 2-3 天：业务

### 必看
- [业务背景](../01-requirements/01-business-context.md)
- [用户角色与场景](../01-requirements/02-user-roles.md)
- [功能需求清单](../01-requirements/03-functional-requirements.md)

### 任务
- [ ] 体验 3 个产品线的现有产品（SaaS / AI / 平台）
- [ ] 观摩 1 次产品评审会
- [ ] 在决策追溯库中查阅过去 1 月 10 个重要决策

## 第 4-5 天：架构

### 必看
- [系统架构](../02-architecture/01-system-architecture.md)
- [技术栈](../02-architecture/02-tech-stack.md)
- [微服务模块](../02-architecture/03-service-modules.md)
- [数据流](../02-architecture/04-data-flow.md)
- [集成模式](../02-architecture/05-integration-patterns.md)
- [术语表](../02-architecture/06-glossary.md)
- [错误码字典](../02-architecture/07-error-codes.md)

### 任务
- [ ] 本地搭建 dev 环境
- [ ] 跑通 1 个核心服务的本地启动
- [ ] 调用 1 个 OpenAPI 接口
- [ ] 在 Grafana 查看 3 个核心服务的健康度

## 第 6-7 天：业务模型

按角色选读：

### 数据 PM / 数据工程
- [SaaS LCM 模型](../03-data-models/01-saas-lcm-model.md)
- [AI 单位经济模型](../03-data-models/02-ai-unit-economics-model.md)
- [平台双边流动性模型](../03-data-models/03-platform-liquidity-model.md)
- [指标字典](../03-data-models/04-metrics-dictionary.md)
- [指标 SQL](../03-data-models/05-metrics-sql.md)
- [数据 Schema](../03-data-models/06-data-schema.md)
- [埋点规范](../03-data-models/07-event-tracking-spec.md)
- [数据治理](../03-data-models/08-data-governance.md)

### 应用开发
- [设计 Tokens](../05-design-system/01-design-tokens.md)
- [组件库](../05-design-system/02-component-library.md)
- [SDK 设计](../11-embedded-components/01-sdk-design.md)
- [组件库设计](../11-embedded-components/02-component-library.md)
- [OpenAPI 规范](../12-api-spec/01-openapi-spec.md)

### AI 工程师
- [助理架构](../07-ai-assistant/01-assistant-architecture.md)
- [30 对话场景](../07-ai-assistant/02-conversation-scenarios.md)
- [智能洞察引擎](../07-ai-assistant/03-intelligent-insight-engine.md)
- [AI 模型治理](../07-ai-assistant/04-model-governance.md)

### 运营
- [Playbook 总览](../06-playbooks/01-playbook-overview.md)
- [37 Playbook 详情](../06-playbooks/02-saas-playbooks.md)
- [实验设计](../06-playbooks/05-experiment-designs.md)
- [触达文案库](../06-playbooks/06-copy-templates.md)
- [用户增长](../09-user-growth/01-growth-loops-segmentation.md)

## 第 8-14 天：实战

### 任务
- [ ] 完成 1 个 onboarding 项目（mentor 安排）
- [ ] 提交第 1 个 PR（小修复或文档完善）
- [ ] 参加 1 次值班 oncall（旁听）
- [ ] 阅读 [运维手册](../14-operations/01-deployment-guide.md) 与 [故障排查](../14-operations/03-troubleshooting.md)
- [ ] 阅读 [测试策略](../15-quality-assurance/01-testing-strategy.md)

## 第 15-30 天：贡献

### 任务
- [ ] 独立完成 1 个 P3 需求
- [ ] 主导 1 次 Code Review
- [ ] 与 1 个业务方接入大使协作
- [ ] 参加 1 次混沌演练

## 30 天考核

mentor 与团队负责人评估：
- 业务理解度
- 技术能力
- 协作能力
- 文化适应

## 常用工具与权限

| 工具 | 用途 | 申请方式 |
|---|---|---|
| GitLab | 代码 | IT 部门 |
| Nacos | 配置 | 中台 PM 审批 |
| Grafana | 监控 | 中台 PM 审批 |
| 决策追溯库 | 决策记忆 | 自助 |
| SkyWalking | 链路追踪 | 中台 PM 审批 |
| ELK / Kibana | 日志 | 中台 PM 审批 |
| Harbor | 镜像仓库 | IT 部门 |
| Vault | 密钥 | 安全部门 |
| Wiki / Confluence | 文档 | 自助 |

## 关键人

| 角色 | 名字 | 联系方式 | 何时联系 |
|---|---|---|---|
| 中台负责人 | XX | @xx | 重大问题升级 |
| 直接 mentor | XX | @xx | 日常 |
| 业务 PM 接口人 | XX | @xx | 业务咨询 |
| 运维 oncall | @oncall | 企微群 | 生产问题 |
| 安全负责人 | XX | @xx | 安全相关 |
| HR | XX | @xx | 人事相关 |

## 文化与价值观

### 中台团队五大原则

1. **决策驱动**：从业务决策反推技术方案，不为技术而技术
2. **数据说话**：所有论断必须有数据支撑
3. **嵌入式思维**：让业务方零摩擦使用，而非"你来用我"
4. **决策留痕**：每个决策都登记，组织记忆 > 个人记忆
5. **失败安全**：复盘对事不对人，鼓励主动暴露问题

### 工作方式

- 双周冲刺（Scrum 轻量版）
- 每天 15min 站会
- 周五下午技术分享
- 月度团建
- 季度 OKR 复盘

## 工作小贴士

- **遇到不会**：先查文档 / 决策追溯库，再问 mentor / 团队
- **遇到 bug**：先排除自己问题，再怀疑系统
- **代码提交**：小步快走，频繁 PR
- **Code Review**：对事不对人，提建议而非指责
- **文档**：写代码同步写文档
- **告警值班**：不要忽略，先评估再行动

## 推荐学习资源

### 业务
- 《用户体验要素》
- 《硅谷增长黑客实战笔记》
- 《SaaS 创业路线图》

### 技术
- 《Spring Boot 实战》
- 《Vue.js 设计与实现》
- 《ClickHouse 原理解析》
- 《Designing Data-Intensive Applications》

### 数据
- 《精益数据分析》
- 《数据中台》
- 《Streaming Systems》

### AI
- 《Prompt Engineering Guide》
- LangChain 官方文档
- OpenAI Cookbook

## 第 1 个月后

- 参与季度 OKR 制定
- 主导小项目
- 建立跨团队联系
- 找到自己的发展方向

## 欢迎加入！

你将参与构建公司未来 5 年最重要的决策基础设施。期待你的贡献。
