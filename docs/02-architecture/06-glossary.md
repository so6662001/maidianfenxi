# 术语表 / 词汇表

> 统一团队对业务、技术、产品概念的认知，避免歧义。

## 业务术语

### 通用

| 术语 | 中文 | 定义 |
|---|---|---|
| Tenant | 租户 | 中台的隔离单元，通常对应一个客户公司 |
| Org / Organization | 企业 / 组织 | 中台用户所属的企业实体 |
| User | 用户 | 终端使用者（区别于业务方系统的用户） |
| Product Line | 产品线 | SaaS / AI / 平台 三大产品线 |
| OneID | 统一身份 | 跨产品识别同一用户/企业的标识 |
| North Star Metric | 北极星指标 | 业务最核心、单一的衡量指标 |

### SaaS

| 术语 | 中文 | 定义 |
|---|---|---|
| MRR | 月经常性收入 | Monthly Recurring Revenue |
| ARR | 年经常性收入 | Annual Recurring Revenue |
| NRR | 净收入留存 | Net Revenue Retention，含扩张 |
| GRR | 毛收入留存 | Gross Revenue Retention，不含扩张 |
| Logo Retention | Logo 留存 | 按客户数计算的留存 |
| LTV | 客户终身价值 | Lifetime Value |
| CAC | 客户获取成本 | Customer Acquisition Cost |
| Payback Period | 回收周期 | 收回 CAC 的月数 |
| Trial | 试用 | 付费前的免费体验期 |
| Onboarding | 新手引导 | 帮助新用户上手 |
| TTV | 首次价值时间 | Time to Value |
| Activation | 激活 | 完成关键动作（如部署 + 首用核心功能） |
| Adoption | 采纳 | 深度使用核心模块 |
| Adoption Score | 采纳深度分 | 综合衡量 |
| Seat | 席位 | 单位用户授权 |
| Expansion | 扩张 | 现有客户加购 |
| Contraction | 收缩 | 现有客户降级 |
| Churn | 流失 | 客户停止使用 |
| Voluntary Churn | 主动流失 | 客户主动取消 |
| Involuntary Churn | 被动流失 | 信用卡失效等 |
| Win-back | 回流 | 流失客户重新付费 |
| NPS | 净推荐值 | Net Promoter Score |
| CSAT | 客户满意度 | Customer Satisfaction |
| CSM | 客户成功经理 | Customer Success Manager |
| Playbook | 运营剧本 | 标准化运营动作流程 |
| PLG | 产品驱动增长 | Product-Led Growth |
| SLG | 销售驱动增长 | Sales-Led Growth |

### AI

| 术语 | 中文 | 定义 |
|---|---|---|
| Token | 令牌 | LLM 处理的基本单位 |
| Prompt | 提示词 | 输入 LLM 的指令 |
| System Prompt | 系统提示词 | 定义 LLM 角色与规则 |
| Few-shot | 少样本提示 | 给 LLM 示例 |
| RAG | 检索增强生成 | Retrieval-Augmented Generation |
| Function Calling | 函数调用 | LLM 主动调用工具 |
| Agent | 智能体 | 多步推理 + 工具调用 |
| Embedding | 向量嵌入 | 文本转向量 |
| LLM | 大语言模型 | Large Language Model |
| TTFV | 首次价值时间 | Time to First Value（AI 上下文） |
| 价值采纳率 | Value Adoption Rate | 输出被复制/导出/继续追问的比例 |
| 单次会话毛利 | Per-session Gross Margin | 收入 - 直接成本 |
| 模型路由 | Model Routing | 按场景选不同模型 |
| 能力缺口 | Capability Gap | 用户需求 vs 模型能力的差距 |

### 平台（双边市场）

| 术语 | 中文 | 定义 |
|---|---|---|
| Buyer | 买家 | 需求侧用户 |
| Merchant | 商家 | 供给侧用户 |
| Inquiry | 询盘 | 买家发出的询价请求 |
| Quote | 报价 | 商家对询盘的报价 |
| Lead | 线索 | 买家留下的潜在商机 |
| Match | 撮合 | 询盘 → 成单 |
| Match Rate | 撮合成功率 | 询盘转化为成单的比例 |
| Liquidity | 流动性 | 双边匹配的活跃度 |
| Network Effect | 网络效应 | 双边规模相互促进 |
| GMV | 成交总额 | Gross Merchandise Value |
| Take Rate | 抽成率 | 平台收入 / GMV |
| 供需比 | Supply/Demand Ratio | 单位类目内 |
| 双边活跃比 | Buyer/Merchant Ratio | DAU 比例 |
| 弹性系数 | Elasticity | 单边增加带来另一边增量 |
| Membership | 会员费 | 商家年度订阅 |
| Traffic Package | 流量包 | 商家购买的曝光额度 |

### 用户增长

| 术语 | 中文 | 定义 |
|---|---|---|
| AARRR | 海盗指标 | Acquisition / Activation / Retention / Revenue / Referral |
| Growth Loop | 增长循环 | 用户行为带来更多用户的闭环 |
| RFM | 用户分层模型 | Recency / Frequency / Monetary |
| LCM | 生命周期管理 | Lifecycle Management |
| Funnel | 漏斗 | 转化路径 |
| Cohort | 队列 | 同期注册用户 |
| K Factor | K 因子 | 病毒式增长系数 |
| Journey | 用户旅程 | 多步触达流程 |
| Segment | 用户分群 | 按条件圈选的用户集 |

### 中台

| 术语 | 中文 | 定义 |
|---|---|---|
| 决策操作系统 | Decision OS | 中台核心范式 |
| 决策追溯库 | Decision Library | 决策与复盘的组织记忆 |
| 战略假设 | Strategic Hypothesis | 长期战略级实验 |
| 决策助理 | Decision Assistant | 对话式 BI |
| 智能洞察 | Intelligent Insight | 异动 + 归因 + 建议 |
| 嵌入式组件 | Embedded Component | 业务系统直接嵌入的能力组件 |
| 数据快照 | Data Snapshot | 决策时刻的数据冻结 |

## 技术术语

### 架构

| 术语 | 中文 | 定义 |
|---|---|---|
| 微服务 | Microservice | 单一职责的服务 |
| DDD | 领域驱动设计 | Domain-Driven Design |
| CDC | 变更数据捕获 | Change Data Capture |
| Event Sourcing | 事件溯源 | 以事件为存储中心 |
| CQRS | 命令查询职责分离 | Command Query Responsibility Segregation |
| Saga | 分布式事务模式 | 长事务补偿模式 |
| Service Mesh | 服务网格 | Istio / Linkerd |
| GitOps | GitOps | 以 Git 为唯一可信源 |

### 数据

| 术语 | 中文 | 定义 |
|---|---|---|
| ODS | 操作数据层 | Raw 原始数据 |
| DWD | 明细数据层 | Detail |
| DWS | 汇总数据层 | Summary |
| ADS | 应用数据层 | Application |
| 物化视图 | Materialized View | 预聚合数据 |
| OLAP | 联机分析处理 | 大数据查询 |
| OLTP | 联机事务处理 | 业务交易 |
| 维度 | Dimension | 分析视角 |
| 度量 | Measure | 数值指标 |
| 血缘 | Lineage | 数据来源追溯 |

### 可观测性

| 术语 | 中文 | 定义 |
|---|---|---|
| SLI | 服务水平指标 | Service Level Indicator |
| SLO | 服务水平目标 | Service Level Objective |
| SLA | 服务等级协议 | 对外承诺 |
| Error Budget | 错误预算 | 1 - SLO |
| APM | 应用性能监控 | Application Performance Monitoring |
| MTBF | 平均无故障时间 | Mean Time Between Failures |
| MTTR | 平均修复时间 | Mean Time To Repair |
| RTO | 恢复时间目标 | Recovery Time Objective |
| RPO | 恢复点目标 | Recovery Point Objective |

### 安全

| 术语 | 中文 | 定义 |
|---|---|---|
| RBAC | 基于角色的访问控制 | Role-Based Access Control |
| ABAC | 基于属性的访问控制 | Attribute-Based |
| JWT | JSON Web Token | 无状态认证 |
| HMAC | 哈希消息认证码 | 签名鉴权 |
| SSO | 单点登录 | Single Sign-On |
| MFA | 多因素认证 | Multi-Factor Authentication |
| PIA | 隐私影响评估 | Privacy Impact Assessment |
| PIPL | 个人信息保护法 | 中国法律 |
| GDPR | 通用数据保护条例 | 欧盟法律 |
| Vault | 密钥管理服务 | HashiCorp Vault |

### 测试

| 术语 | 中文 | 定义 |
|---|---|---|
| SAST | 静态应用安全测试 | Static Application Security Testing |
| DAST | 动态应用安全测试 | Dynamic |
| SCA | 软件组成分析 | 依赖漏洞扫描 |
| E2E | 端到端测试 | End-to-End |
| 单元测试 | Unit Test | 函数/类级 |
| 集成测试 | Integration Test | 服务间联调 |
| 性能测试 | Performance Test | 压测 |
| 混沌工程 | Chaos Engineering | 故障注入 |
| 契约测试 | Contract Test | API 契约 |

### 部署

| 术语 | 中文 | 定义 |
|---|---|---|
| 蓝绿发布 | Blue-Green Deployment | 两套环境切换 |
| 金丝雀发布 | Canary Release | 渐进式发布 |
| 灰度发布 | Gray Release | 按比例分流 |
| 滚动更新 | Rolling Update | 逐步替换 |
| HPA | 水平自动扩缩 | Horizontal Pod Autoscaler |
| PDB | Pod 中断预算 | Pod Disruption Budget |

## 缩写表

| 缩写 | 全称 |
|---|---|
| API | Application Programming Interface |
| SDK | Software Development Kit |
| DSL | Domain Specific Language |
| ORM | Object-Relational Mapping |
| MQ | Message Queue |
| KV | Key-Value |
| QPS | Queries Per Second |
| TPS | Transactions Per Second |
| RPS | Requests Per Second |
| CRUD | Create Read Update Delete |
| BI | Business Intelligence |
| ETL | Extract Transform Load |
| ELT | Extract Load Transform |
| OKR | Objectives and Key Results |
| KPI | Key Performance Indicator |
| PM | Product Manager |
| PRD | Product Requirements Document |
| MVP | Minimum Viable Product |
| CSM | Customer Success Manager |
| SRE | Site Reliability Engineering |
| DevOps | Development + Operations |
| CI/CD | Continuous Integration / Delivery |

## 内部约定

| 约定 | 说明 |
|---|---|
| 中台 | 指本"用户分析中台 + 决策操作系统" |
| 业务方 | 接入中台的业务系统 / 团队 |
| 总产品经理 | 集团级产品总负责人，本中台核心用户 |
| MVP | 首批上线交付物 |
| P0 | 必须上线 |
| P1 | 强烈推荐上线 |
| P2 | 可选 |
| P3 | 长期规划 |

## 维护

- 新术语先在此处定义后再使用
- 季度评审，淘汰废弃术语
- 跨团队评审，确保理解一致
