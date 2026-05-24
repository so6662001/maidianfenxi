# 用户分析中台 · 决策操作系统

> 一个面向**多产品集团**（SaaS / AI / 平台型）的「用户分析 + 决策操作系统 + 用户增长引擎」三位一体中台。

## 项目定位

不是传统 BI 报表系统，而是**对话式决策助理 + 嵌入式能力组件**的"决策操作系统"。

核心能力：

- 5 张战略决策看板（产品组合、健康度、PM 效能、战略假设、决策待办池）
- 3 套业务模型（SaaS LCM / AI 单位经济 / 平台双边流动性）
- 智能洞察引擎（异动检测 + AI 归因 + 决策建议）
- 决策追溯库（组织决策记忆与复盘）
- 37 个用户增长 Playbook
- 对话式决策助理（30+ 高频场景）
- 嵌入式能力组件（嵌入业务系统，开箱即用）

## 技术栈

- **后端**：Java 17 + Spring Boot 3.2 + Spring Cloud Alibaba 2023 + MyBatis-Plus + ClickHouse + MySQL + Redis + Kafka + Flink
- **前端**：Vue 3.4 + TypeScript 5 + Vite 5 + Pinia + Ant Design Vue 4 + ECharts 5 + wujie（微前端）
- **AI**：LLM + RAG + Function Calling
- **基础设施**：K8s + Nacos + Sentinel + SkyWalking + ELK

## 文档导航

完整设计文档位于 [`docs/`](./docs/) 目录，开发提示词位于 [`docs/prompts/`](./docs/prompts/)。

**开发起点**：阅读 [`docs/README.md`](./docs/README.md) 总索引。

## 落地建议

```
MVP（建议先做）：
  1. S-1 产品组合矩阵（让总产品经理立刻用上）
  2. <UserProfile360 /> 嵌入组件（让业务方立刻接入）
  3. 决策助理 L1-L3（让全员尝鲜）
  4. 决策追溯库（先手动登记）

V1：智能洞察 + 8 个 P0 Playbook 上线
V2：3 套业务模型看板全量
V3：智能助理 L4-L6 + 自适应学习
```

## License

Internal Use Only
