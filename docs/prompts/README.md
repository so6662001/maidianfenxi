# AI 开发提示词 · 使用指南

> 本目录的每个 `.md` 文件都是一份**完整的 AI 编码助手提示词**，可直接复制到 Cursor / Claude / ChatGPT / Copilot Chat 等工具中使用。

## 📋 设计原则

每份提示词严格按 **"角色 + 目标 + 上下文 + 约束 + 交付物 + 验收标准 + 安全要求"** 7 段结构编写，目标：

- ✅ **不能缺少功能**：每份提示词列出全部必需功能
- ✅ **确保逻辑完整**：明确"输入 → 处理 → 输出"链路
- ✅ **代码符合规范**：内含编码规范、命名规范、文档规范
- ✅ **安全性能达标**：定义 SLA、限流、鉴权、审计要求

## 🗂️ 提示词清单

### 项目级
- [00 - 总纲：项目脚手架与统一规范](./00-master-prompt.md) ⭐ 必读
- [01 - 后端 Spring Boot 项目初始化](./01-backend-setup.md)
- [02 - 前端 Vue3 项目初始化](./02-frontend-setup.md)

### SDK 类
- [03 - Java 数据上报 SDK](./03-java-sdk.md)
- [04 - Vue3 数据上报 SDK](./04-vue-sdk.md)

### 核心服务
- [05 - 数据采集服务 Collector](./05-collector-service.md)
- [06 - 指标平台服务 Metric](./06-metric-platform.md)
- [07 - 分群引擎服务 Segment](./07-segment-service.md)
- [08 - Journey 与触达服务 Reach](./08-reach-service.md)
- [09 - AB 实验平台 Experiment](./09-experiment-platform.md)
- [10 - 决策追溯库 Decision](./10-decision-library.md)

### AI 应用
- [11 - 决策助理 Assistant](./11-ai-assistant.md)
- [12 - 智能洞察引擎 Insight](./12-insight-engine.md)

### 前端
- [13 - S-1 产品组合矩阵看板](./13-dashboard-s1.md)
- [14 - S-2 至 S-5 战略看板](./14-dashboard-s2-s5.md)
- [15 - B-1 至 B-3 业务模型看板](./15-dashboard-business-models.md)
- [16 - Vue3 嵌入式组件库](./16-embedded-components.md)

### 平台能力
- [17 - OpenAPI 服务](./17-openapi-service.md)

### DevOps & 质量
- [18 - CI/CD 与质量门禁](./18-cicd-quality-gate.md)
- [19 - 安全合规审计](./19-security-compliance.md)

## 🚀 使用方法

### 方式 1：单模块开发

```
1. 选定要开发的模块（如：指标平台）
2. 打开 06-metric-platform.md
3. 全文复制到 AI 编码助手
4. AI 输出完整代码
5. Review + 调试 + 提交
```

### 方式 2：跨模块协作

```
1. 先用 00-master-prompt.md 让 AI 理解整体范式
2. 再用 03-java-sdk.md 等子模块提示词
3. AI 会自动遵循总纲规范
```

### 方式 3：增量开发

```
1. 先用提示词生成骨架
2. 后续增量需求 = "在 XX 项目（已按提示词生成）基础上，新增/修改 YY 功能"
3. AI 自动遵循原项目规范
```

## ⚠️ 注意事项

1. **提示词不是一次性消耗品**：每次需求变更可以重新使用
2. **AI 生成的代码必须 Review**：尤其是安全敏感（鉴权 / SQL / 加密）部分
3. **本地测试通过再提交**：单元测试 + 集成测试
4. **遵循 PR 流程**：分支 → 评审 → 合并
5. **更新提示词**：如发现 AI 持续犯同类错误，更新对应提示词

## 📞 反馈与迭代

- 提示词使用过程中发现问题 → 反馈到中台 PM
- 提示词季度评审一次，持续优化
- 优秀的代码生成案例 → 沉淀为提示词附录

## 🔗 相关文档

- 设计文档总索引：[../README.md](../README.md)
- 技术架构：[../02-architecture/](../02-architecture/)
- 数据模型：[../03-data-models/](../03-data-models/)
- 看板设计：[../04-dashboards/](../04-dashboards/)
