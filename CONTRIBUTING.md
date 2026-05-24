# 贡献指南

欢迎为"用户分析中台 · 决策操作系统"项目贡献代码与文档！

## 行为准则

我们承诺为所有贡献者提供友好、安全、包容的环境。详见 `CODE_OF_CONDUCT.md`（如有）。

## 如何贡献

### 1. 报告问题（Issue）

发现 Bug 或有建议？请使用对应模板：

- **Bug Report**：复现步骤、期望结果、实际结果、环境
- **Feature Request**：业务背景、期望功能、价值评估
- **Documentation**：哪里不清楚或错误

### 2. 提交代码（Pull Request）

#### 流程

```
1. Fork（外部）/ Clone（内部）
2. 创建 feature 分支（feature/xxx 或 fix/xxx）
3. 开发 + 测试
4. 提交（遵循 Conventional Commits）
5. 推送
6. 创建 PR
7. CI 通过 + Code Review
8. 合并
```

#### 分支命名

```
main                       # 生产
release/v1.0.0             # 预发
develop                    # 开发集成
feature/<jira-key>-<slug>  # 新功能
fix/<jira-key>-<slug>      # Bug 修复
hotfix/<slug>              # 紧急修复
docs/<slug>                # 文档
chore/<slug>               # 杂项
```

#### 提交规范（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

<footer>
```

type 限制：
- `feat`: 新功能
- `fix`: Bug 修复
- `docs`: 文档变更
- `style`: 格式化
- `refactor`: 重构
- `perf`: 性能优化
- `test`: 测试
- `build`: 构建系统
- `ci`: CI 配置
- `chore`: 杂项
- `revert`: 回滚

示例：
```
feat(metric): 支持 NRR 指标查询

- 新增 M-SE-001 NRR 指标
- 支持按行业、客单价段、销售年份维度
- 添加物化视图加速

Closes #123
```

#### PR 模板

详见 `.github/PULL_REQUEST_TEMPLATE.md` 或 `.gitlab/merge_request_templates/`。

#### Code Review

- 至少 1 人 Approve
- 安全敏感（鉴权 / 加密 / SQL）必须 2 人 Approve
- 大变更（> 500 行）拆分 PR

### 3. 编写文档

#### 文档位置

- 设计文档：`docs/0X-xxx/`
- API 文档：随代码（SpringDoc 自动生成）
- 组件文档：`docs/11-embedded-components/` + VitePress

#### 文档规范

- Markdown 格式
- 中文优先（含英文术语 + 缩写）
- 章节层级清晰
- 代码块标注语言
- 表格用于结构化对比
- 图用 Mermaid（避免外部图床）

## 编码规范

### Java 后端

遵循阿里巴巴 Java 开发手册（黄山版）+ 项目特有约定（见 [docs/prompts/00-master-prompt.md](./docs/prompts/00-master-prompt.md)）。

关键：
- Java 17
- 4 空格缩进
- UTF-8
- 构造函数注入 + Lombok
- 全局异常处理
- SQL 必须参数化
- 单测覆盖率 ≥ 60%（核心 80%）

### TypeScript / Vue3 前端

- TypeScript strict mode
- ESLint + Prettier + Stylelint（CI 强制）
- 组件 PascalCase
- 禁止 any（除非显式注释）
- 单测覆盖率 ≥ 60%（核心 80%）

### 数据库

- 表名 snake_case，前缀 `t_`
- 字段 snake_case
- 主键 `id` bigint
- 必备字段：`tenant_id`, `gmt_create`, `gmt_modified`, `creator_id`, `modifier_id`, `is_deleted`, `version`
- 索引必加
- SQL 必须 EXPLAIN

### Git Hooks

启用 Husky + lint-staged：
- pre-commit: ESLint / Prettier / Stylelint / Checkstyle
- commit-msg: commitlint
- pre-push: 测试

## 测试要求

### 单元测试

每个新功能必须有单元测试。覆盖率门禁：
- 整体 ≥ 60%
- 核心模块 ≥ 80%
- 关键安全函数 100%

### 集成测试

涉及多个服务联调的功能必须有集成测试。

### E2E 测试

涉及关键业务路径的功能必须更新 E2E 测试。

详见 [测试策略](./docs/15-quality-assurance/01-testing-strategy.md)。

## 安全要求

- 永远不提交密码 / Token / AppSecret 到 Git
- 使用 `.gitignore` 排除敏感文件
- 用 Vault / Sealed Secrets 管理密钥
- PR 自动安全扫描，无 Critical 漏洞才能合并
- 涉敏功能必须做 PIA 评估

详见 [安全合规](./docs/12-api-spec/02-authentication.md)。

## 性能要求

详见 [SLO 文档](./docs/14-operations/04-slo-sli.md) 和 [性能优化指南](./docs/15-quality-assurance/02-performance-optimization.md)。

## 文档同步

代码变更必须同步：
- API 变更 → 更新 OpenAPI 文档
- 数据库变更 → 更新 Schema 文档 + Flyway 脚本
- 看板变更 → 更新 UI Spec
- 新指标 → 注册到元数据中心
- Playbook 变更 → 更新 Playbook 文档

## 重大变更

详见 [变更管理流程](./docs/13-okr-governance/02-change-management.md)。

L1/L2 变更必须 RFC + 评审委员会。

## 决策留痕

任何重要技术决策必须登记到决策追溯库。

```
- 架构选型决策
- 重大重构决策
- 性能优化决策
- 安全策略决策
```

## 发布流程

详见 [部署指南](./docs/14-operations/01-deployment-guide.md) 和 [CI/CD 提示词](./docs/prompts/18-cicd-quality-gate.md)。

灰度发布原则：10% → 50% → 100%

## 紧急流程

生产故障：详见 [故障排查](./docs/14-operations/03-troubleshooting.md)。

## 帮助与支持

- 文档：`docs/`
- Slack / 企微：`#analytics-platform-dev`
- Mentor：见 [新人入职](./docs/00-overview/04-onboarding.md)
- 紧急联系：`@analytics-oncall`

## 许可证

Internal Use Only

---

感谢你的贡献！
