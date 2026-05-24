# 非功能性需求

## 一、性能需求

### 响应时间
| 场景 | 指标 | 目标 |
|---|---|---|
| 看板首屏加载 | P95 | ≤ 2s |
| 单个图表查询 | P95 | ≤ 5s |
| 即席查询（DSL） | P95 | ≤ 8s（超 30s 自动 kill） |
| OpenAPI 简单查询 | P95 | ≤ 500ms |
| 数据采集接口 | P99 | ≤ 50ms |
| 决策助理简单查询 | P95 | ≤ 2s |
| 决策助理复杂分析 | P95 | ≤ 8s（含 LLM） |
| 嵌入式组件初次渲染 | P95 | ≤ 1.5s |

### 吞吐量
| 服务 | 目标 |
|---|---|
| 数据采集服务（单 Pod 4C8G） | ≥ 1 万 events/s |
| 指标查询服务 | ≥ 500 QPS |
| OpenAPI 网关 | ≥ 2000 QPS |
| Kafka 总吞吐 | ≥ 10 万 events/s |
| ClickHouse 单节点 | 1 亿行查询 ≤ 1s |

### 缓存命中率
| 缓存 | 目标 |
|---|---|
| Redis 元数据缓存 | ≥ 99% |
| 指标查询结果缓存 | ≥ 70% |
| 物化视图命中率 | ≥ 60% |
| LLM Function 调用结果缓存 | ≥ 50% |

## 二、可用性

| 服务 | SLA |
|---|---|
| 数据采集服务 | ≥ 99.99% |
| OpenAPI 网关 | ≥ 99.95% |
| 看板服务 | ≥ 99.9% |
| 决策助理 | ≥ 99.5% |
| Journey 触达 | ≥ 99.9% |

### 容灾
- 主备双 K8s 集群
- ClickHouse 副本数 ≥ 2
- Kafka 分区副本 ≥ 3
- 数据快照永久存档（OSS 跨区备份）

## 三、安全需求

### 鉴权
- 内部用户：SSO + JWT，Token 1h 过期 + Refresh Token
- 业务系统 API：AppKey + AppSecret + HMAC-SHA256 + 时间戳防重放
- 嵌入组件：业务后端签发短期 Token（≤ 15 min），前端不存 Secret

### 授权
- RBAC：角色 → 资源 → 操作 三级
- 行级权限：所有查询自动加 tenant_id / org_id / product_line 过滤
- 字段级脱敏：手机号 / 身份证 / 订单金额 按角色脱敏

### 防攻击
- 限流：Sentinel，按 AppKey/IP/接口 多维度
- 防重放：Nonce + 5min 时间窗 Redis 缓存
- 防注入：所有 SQL 走参数化；所有用户输入校验
- XSS：前端默认转义；后端响应头 Content-Security-Policy
- CSRF：所有写操作 Token 校验
- 接口幂等：写接口 X-Idempotency-Key 必填

### 审计
- 所有数据访问留痕（who / when / what）
- 所有数据导出留痕 + 水印
- 所有决策修改留痕
- 留痕日志保留 ≥ 180 天

### 合规
- 个人信息保护法（PIPL）合规
- 数据安全法合规
- 涉及海外用户：GDPR 评估
- 数据出境：跨境传输合规评估
- 用户授权链路完整
- 提供用户数据导出 + 删除接口（GDPR Right to Access/Delete）

### 加密
- 数据库静态加密：敏感字段（手机/身份证/金额）字段级加密
- 传输加密：全链路 HTTPS / TLS 1.3
- 密钥管理：HashiCorp Vault / KMS
- 配置敏感项：Nacos + Vault 集成

## 四、可扩展性

### 水平扩展
- 所有应用服务无状态，可 HPA 弹性扩缩
- 数据库读写分离，ClickHouse 分片
- Kafka 按业务/AppKey 分区

### 多租户
- 数据物理隔离方案：tenant_id 字段 + 行级权限（初期）
- 高敏租户独立 Schema（演进期）

### 国际化
- 前端 vue-i18n，至少 zh-CN / en-US
- 时间字段统一 UTC 存储，前端按用户时区显示

## 五、可观测性

### 监控
- APM：SkyWalking（链路追踪 + 性能监控）
- Metrics：Prometheus + Grafana
- Logs：Filebeat → Kafka → Elasticsearch → Kibana
- 业务监控：自研业务大盘

### 告警
- 服务告警：SLA / 错误率 / 延迟（Alertmanager）
- 业务告警：智能洞察引擎产出
- 通道：飞书 / 钉钉 / 企微 / 邮件 / 短信

### TraceID
- 全链路 traceId 透传（前端 → 网关 → 服务 → SDK → DB）
- 日志、监控、告警通过 traceId 关联

## 六、可维护性

### 代码规范
- 后端：阿里巴巴 Java 开发手册（黄山版）
- 前端：ESLint + Prettier + Stylelint，自定义规则集
- Git：Conventional Commits + Husky + lint-staged + commitlint
- 强制 Code Review：所有 PR 至少 1 人 Approve

### 测试覆盖
- 单元测试覆盖率 ≥ 60%
- 关键模块（鉴权 / 指标计算 / 决策追溯）≥ 80%
- 接口契约测试（Pact / Spring Cloud Contract）
- E2E 测试覆盖关键业务路径

### 文档
- 代码注释：复杂逻辑必须注释"为什么"而非"做什么"
- API 文档：SpringDoc OpenAPI 自动生成
- 组件文档：VitePress Storybook
- 运维手册：部署 / 回滚 / 故障排查

### 灰度发布
- 蓝绿 / 金丝雀：Argo Rollouts
- 一键回滚：保留近 5 个版本
- 配置中心：Nacos 命名空间隔离 dev/test/staging/prod

## 七、合规与隐私

### 数据生命周期
| 数据类型 | 热存 | 冷存 | 归档/删除 |
|---|---|---|---|
| 行为明细（dwd） | 24 个月 | 36 个月 | 删除 |
| 聚合数据（dws） | 36 个月 | 永久 | — |
| 元数据 | 永久 | — | — |
| 决策记录 | 永久 | — | — |
| 操作日志 | 6 个月 | 24 个月 | 删除 |
| LLM 对话日志 | 3 个月 | 12 个月 | 删除（含 PII 脱敏） |

### 用户隐私
- 注册时明确告知数据使用范围 + 同意机制
- 提供"数据导出" + "数据删除"自助入口
- AI 产品对话默认不参与训练，明确开关
- 敏感数据脱敏存储

## 八、容量规划

| 资源 | 初始 | 1 年后预估 |
|---|---|---|
| 行为事件量 | 100 万/天 | 1 亿/天 |
| MySQL 存储 | 100 GB | 1 TB |
| ClickHouse 存储 | 1 TB | 50 TB |
| Redis 内存 | 32 GB | 256 GB |
| Kafka 总吞吐 | 1 万/s | 50 万/s |
| LLM 调用 | 1 万/天 | 100 万/天 |
| 接入业务系统 | 2 个 | 15 个 |
