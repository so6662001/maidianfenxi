# 技术栈选型

> 原则：与现有公司技术栈一致（Java + Vue3），优先成熟稳定的开源方案。

## 一、后端

### 基础
| 类别 | 选型 | 版本 | 备注 |
|---|---|---|---|
| 语言 | Java | 17 (LTS) | 与现有系统一致 |
| 框架 | Spring Boot | 3.2.5 | 主流稳定 |
| 微服务 | Spring Cloud | 2023.0.1 | |
| 微服务套件 | Spring Cloud Alibaba | 2023.0.1.0 | Nacos / Sentinel / Seata |
| 构建 | Maven | 3.9+ | 与公司一致 |
| JDK | Eclipse Temurin | 17 | OpenJDK 发行版 |

### 中间件
| 类别 | 选型 | 版本 | 用途 |
|---|---|---|---|
| 注册中心 | Nacos | 2.3+ | 服务注册 + 配置中心 |
| 限流 | Sentinel | 1.8+ | 网关限流 + 熔断 |
| 网关 | Spring Cloud Gateway | — | 统一鉴权 + 路由 |
| ORM | MyBatis-Plus | 3.5.5 | 国内最熟悉 |
| 数据源 | Dynamic Datasource + Druid | — | 多数据源 + 连接池 |
| 缓存 | Redis + Redisson | 7.x / 3.27 | 分布式锁/缓存 |
| MQ | Apache Kafka | 3.6+ | 行为事件高吞吐 |
| MQ | RocketMQ | 5.x | 业务事件可选 |
| 流计算 | Apache Flink | 1.18+ | 实时指标/标签 |
| 调度 | XXL-Job | 2.4+ | 离线任务调度 |
| API 文档 | SpringDoc OpenAPI | 2.x | 自动生成 OpenAPI 3.0 |

### 存储
| 类别 | 选型 | 用途 |
|---|---|---|
| 关系数据库 | MySQL 8.0 | 元数据 + 业务数据 |
| OLAP | ClickHouse 23.x | 行为明细 + 即席查询 |
| 缓存 | Redis 7 | 缓存 + 标签 |
| 搜索 | Elasticsearch 8.x | 全文搜索 + 日志 |
| 文件 | MinIO / 阿里云 OSS | 报表导出 + 决策快照 |

### 监控可观测
| 类别 | 选型 |
|---|---|
| APM | SkyWalking 9.x |
| Metrics | Prometheus + Grafana |
| Logs | Filebeat + ELK / Loki |
| 链路追踪 | OpenTelemetry / SkyWalking |

### AI / LLM
| 类别 | 选型 | 备注 |
|---|---|---|
| LLM 接入 | LangChain4j 或自封装 | 多 Provider 兼容 |
| 主模型 | OpenAI GPT-4 / Claude / 通义千问 | 按场景路由 |
| 小模型 | GPT-4o-mini / Claude Haiku / Qwen-Turbo | 简单请求 |
| RAG | LangChain4j + Embedding | 检索元数据 |
| 向量库 | Milvus / Qdrant | RAG 检索 |
| Embedding | text-embedding-3-small | OpenAI |

## 二、前端

### 基础
| 类别 | 选型 | 版本 |
|---|---|---|
| 框架 | Vue | 3.4+ |
| 语言 | TypeScript | 5+ |
| 构建 | Vite | 5+ |
| 包管理 | pnpm | 8+ |
| 状态 | Pinia | 2+ |
| 路由 | Vue Router | 4+ |

### UI
| 类别 | 选型 | 备注 |
|---|---|---|
| 主 UI 库 | Ant Design Vue | 4.x，企业级中后台最完善 |
| 图表 | Apache ECharts | 5+ |
| 拖拽布局 | Vue-Grid-Layout | 自助看板 |
| Markdown | markdown-it / shiki | 决策助理输出渲染 |
| 表单 | Ant Design Form + vee-validate | |
| 图标 | @ant-design/icons-vue | |

### 微前端
| 类别 | 选型 | 备注 |
|---|---|---|
| 主框架 | wujie | 推荐：基于 Web Components，Vue3 + Vite 友好 |
| 备选 | qiankun | 老业务系统可能用到 |

### 工程化
| 类别 | 选型 |
|---|---|
| Lint | ESLint + Prettier + Stylelint |
| Git Hooks | Husky + lint-staged + commitlint |
| 测试 | Vitest + Vue Test Utils + Playwright |
| 组件文档 | VitePress + Vue Demi |

### HTTP / 请求
| 类别 | 选型 |
|---|---|
| HTTP 库 | Axios |
| WebSocket | 原生 WebSocket + 心跳 |
| Mock | MSW / vite-plugin-mock |

## 三、SDK

### Java SDK
| 类别 | 选型 |
|---|---|
| 基础依赖（最小化） | OkHttp 4 + Jackson + SLF4J |
| Spring Boot Starter | analytics-spring-boot-starter |
| 本地队列 | Chronicle Queue / 自封 RandomAccessFile |
| 字节码增强 | Byte Buddy（可选，AOP 实现 @Track） |

### Vue3 SDK
| 类别 | 选型 |
|---|---|
| 构建 | Vite Library Mode |
| 离线存储 | idb-keyval（IndexedDB 封装） |
| 体积 | gzip ≤ 30KB |

## 四、部署与运维

| 类别 | 选型 |
|---|---|
| 容器 | Docker (multi-stage build) |
| 编排 | Kubernetes 1.28+ |
| 包管理 | Helm 3 |
| GitOps | Argo CD |
| CI/CD | GitLab CI 或 Jenkins |
| 镜像仓库 | Harbor |
| 密钥管理 | HashiCorp Vault 或云厂商 KMS |

## 五、数据治理

| 类别 | 选型 |
|---|---|
| 元数据管理 | DataHub 或自研 |
| 数据质量 | Great Expectations 思想 + 自研规则 |
| 调度 | XXL-Job + Apache DolphinScheduler（可选） |

## 六、与现有系统的对接

| 现有系统 | 对接方式 |
|---|---|
| 公司 IAM / SSO | OAuth2 / OIDC 接入 |
| 企微 / 钉钉 | 应用集成 + Bot Webhook |
| 邮件 | SMTP 或第三方（SendGrid / 阿里云邮件） |
| 短信 | 阿里云短信 / 腾讯云短信 |
| Push | 极光 / 个推 / Firebase |

## 七、技术选型理由备忘

### 为什么选 ClickHouse 而非 Doris/StarRocks？
- 社区成熟，国内案例多（神策/字节用同款）
- 单机性能极强，运维门槛适中
- 与 Java 生态集成成熟（clickhouse-jdbc）

### 为什么选 Ant Design Vue 而非 Element Plus？
- 中后台场景下 Ant Design 更专业
- 表格 / 表单 / 树组件能力更强
- 配套的 Pro Components 加速开发

### 为什么选 wujie 而非 qiankun？
- 基于 Web Components 隔离更彻底
- 对 Vue3 + Vite 支持更好
- 性能优于 qiankun
- 老业务系统兼容场景仍可备用 qiankun

### 为什么用 Spring Cloud Alibaba 而非纯 Spring Cloud？
- Nacos 在国内运维更友好
- Sentinel 限流能力强，控制台完善
- Seata 分布式事务（如需）

### 为什么主模型用 LangChain4j？
- Java 原生 LLM 框架，避免跨语言调用
- 支持多 Provider 切换
- RAG / Function Calling / Agent 一站式
