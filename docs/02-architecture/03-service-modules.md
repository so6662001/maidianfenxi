# 微服务模块划分

> 按 DDD 限界上下文拆分。**初期合并为 4-5 个服务，跑顺再拆。**

## 一、完整微服务清单（V2 目标）

| 服务名 | 职责 | 端口 | 说明 |
|---|---|---|---|
| `analytics-gateway` | API 网关、鉴权、限流、路由 | 8000 | 入口 |
| `analytics-auth` | OneID、SSO、RBAC、租户 | 8001 | 身份 |
| `analytics-collector` | 数据采集（高吞吐） | 8002 | 入数据 |
| `analytics-meta` | 事件/属性/指标/数据源元数据 | 8003 | 治理 |
| `analytics-metric` | 指标平台（定义+查询） | 8004 | 核心 |
| `analytics-query` | 即席查询（漏斗/留存/路径） | 8005 | 分析 |
| `analytics-tag` | 标签平台 + 用户画像 | 8006 | 画像 |
| `analytics-segment` | 分群引擎 | 8007 | 圈人 |
| `analytics-reach` | Journey 编排 + 触达中心 | 8008 | 运营 |
| `analytics-experiment` | AB 实验平台 | 8009 | 实验 |
| `analytics-report` | 看板 / 报表 / 订阅 / 导出 | 8010 | 输出 |
| `analytics-openapi` | 对外开放 API 编排 | 8011 | 输出 |
| `analytics-admin` | 管理后台 BFF | 8012 | 后台 |
| `analytics-job` | 调度任务（XXL-Job Executor） | 8013 | 离线 |
| `analytics-assistant` | 决策助理（LLM 应用） | 8014 | AI |
| `analytics-insight` | 智能洞察引擎（异动+归因+建议） | 8015 | AI |
| `analytics-decision` | 决策追溯库 | 8016 | 沉淀 |

## 二、MVP 阶段合并方案（建议）

```
4 个核心服务：
├─ analytics-gateway       (网关)
├─ analytics-data          (采集 + Meta + 指标 + 标签 + 分群合并)
├─ analytics-app           (Journey + AB + 看板 + 决策追溯合并)
└─ analytics-assistant     (助理 + 洞察引擎)
```

跑顺后按需拆分。

## 三、服务边界与依赖

### 上下游依赖图

```
[业务系统] ──┐
             ├─► gateway ──► auth ───┐
             │                       │
[SDK] ───────► collector ──► Kafka   │
                              │      ▼
                              ├─► meta ──┐
                              │          ▼
                              ▼      metric ◄── query ◄── report
                          ClickHouse           │
                              ▲               ▼
                              │           segment ◄── tag
                              │               │
                              │               ▼
                              │           reach ◄── experiment
                              │               │
                              │               ▼
                              │       decision ◄── insight
                              │               │
                              └───────────────┴── assistant ──► [用户]
```

## 四、各服务详细职责

### analytics-gateway
- 路由：根据 URL 路由到下游服务
- 鉴权：JWT 验证、AppKey HMAC 验证
- 限流：Sentinel 按 IP / 用户 / AppKey / 接口 多维度
- 日志：traceId 注入、请求/响应日志
- 跨域：CORS 处理
- 灰度路由：根据 Header / 用户标识灰度分流

### analytics-auth
- 用户登录（SSO 集成）
- JWT 签发与刷新
- OneID 管理（设备/账号/企业映射）
- RBAC 角色权限
- 多租户隔离
- 审计日志记录
- 用户偏好（看板订阅、收藏）

### analytics-collector
- 接收 SDK 上报事件
- 数据校验（必填、类型、事件注册检查）
- 字段丰富（server_time、geo、ua 解析）
- 写 Kafka（按 app_id 分区）
- 失败 DLQ
- 监控指标暴露
- **完全无状态**

### analytics-meta
- 事件元数据（事件 code、参数、责任人）
- 指标元数据（指标定义、口径、版本）
- 数据源元数据（表、字段、血缘）
- 标签元数据（标签定义、生产方式）
- 元数据评审与版本管理

### analytics-metric
- 指标 DSL 解析（JSON DSL → Calcite → SQL）
- 物化视图自动命中
- 查询结果缓存（Redis）
- 慢查询保护（超时 kill）
- 行级权限注入
- 查询审计

### analytics-query
- 漏斗分析
- 留存矩阵
- 路径分析（桑基）
- 行为序列挖掘
- Session Replay 查询

### analytics-tag
- 离线标签调度（XXL-Job 触发 Spark/SQL）
- 实时标签更新（消费 Kafka）
- 标签查询 API（Redis + ClickHouse 双查）
- 用户画像聚合接口

### analytics-segment
- 分群规则配置
- 分群人数实时预估
- 分群快照（每日）
- 分群导出（CSV / Kafka / API）
- 分群血缘

### analytics-reach
- Journey 编排（节点+流程图）
- 触发条件评估
- 通道适配（站内信/邮件/短信/Push/企微）
- 任务调度与重试
- 疲劳度控制（用户级）
- 退订管理
- 效果追踪

### analytics-experiment
- 实验配置
- 流量分配（哈希分流）
- SRM 检查
- 显著性计算
- 分群下钻
- 实验归档

### analytics-report
- 战略看板配置与查询编排
- 业务模型看板
- 自助看板
- 看板订阅（邮件/企微）
- 报表导出（PDF / Excel / 图片）

### analytics-openapi
- 对外 API 编排
- 多语言 SDK Client 生成
- 限流（按 AppKey）
- 幂等控制
- 沙箱环境

### analytics-admin
- 管理后台 BFF
- 看板配置后台
- 用户/权限/租户管理
- 元数据管理后台
- 系统配置

### analytics-job
- XXL-Job Executor
- 离线指标计算
- 标签生产
- 数据质量检查
- 数据归档

### analytics-assistant
- 自然语言意图识别
- Function Calling 工具集
- RAG 检索（元数据 + 决策追溯）
- 多轮上下文管理
- LLM 调用与降级
- 权限校验
- 操作二次确认
- 反馈收集

### analytics-insight
- 异动检测调度
- 多算法策略（阈值/STL/Prophet/Isolation Forest）
- 维度归因（Adtributor）
- LLM 摘要与建议生成
- 推送编排
- 反馈学习

### analytics-decision
- 决策登记（5 模板）
- 数据快照冻结
- 自动复盘调度
- 知识沉淀（标签 + 检索）
- 与 PM 效能榜联动
- 与战略假设联动

## 五、服务通信

### 同步
- 服务间：HTTP REST + Feign（Spring Cloud OpenFeign）
- 外部：HTTPS

### 异步
- 主：Kafka（业务事件 + 行为事件）
- 备：RocketMQ（事务消息 + 顺序消息场景）

### RPC
- 不引入 Dubbo / gRPC，统一 HTTP，简化技术栈

### 服务发现
- Nacos

### 配置中心
- Nacos 配置 + Vault（敏感配置）

## 六、服务规范

| 项 | 规范 |
|---|---|
| 命名 | `analytics-{module}`，全小写 |
| 端口 | 业务端口 8000-8099，管理端口 +1000 |
| 健康检查 | `/actuator/health` 必须 |
| 监控 | `/actuator/prometheus` 必须 |
| 文档 | `/swagger-ui.html` 必须 |
| 日志格式 | JSON，含 traceId |
| 配置 | application-{env}.yml + Nacos 远程 |
| 镜像 | 多阶段构建，distroless 或 jre-jammy |
| K8s | Deployment + Service + HPA + PDB |
