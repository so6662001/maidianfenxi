# 部署指南

## 一、环境分层

| 环境 | 域名 | 用途 | 数据 |
|---|---|---|---|
| dev | dev.analytics.internal | 开发联调 | Mock + 少量真实 |
| test | test.analytics.internal | 测试团队 | 测试数据 |
| staging | staging.analytics.yourcompany.com | 预发布 | 生产脱敏 |
| sandbox | sandbox.analytics.yourcompany.com | 业务方接入测试 | 隔离测试数据 |
| prod | api.analytics.yourcompany.com | 生产 | 真实 |

## 二、K8s 集群规划

### 命名空间
```
analytics-dev
analytics-test
analytics-staging
analytics-sandbox
analytics-prod
analytics-monitoring (Prometheus, Grafana, AlertManager)
analytics-logging (Elasticsearch, Kibana, Filebeat)
analytics-data (ClickHouse, Kafka, MySQL, Redis - 通常托管在外部)
```

### 节点池
| 节点池 | 规格 | 用途 |
|---|---|---|
| general | 8C32G ×N | 应用服务 |
| compute | 16C64G ×N | Flink / Spark |
| memory | 8C64G ×N | Redis / ClickHouse |
| ai | GPU 节点 ×N | LLM 推理（如自部署） |

## 三、部署清单（按依赖顺序）

```
阶段 1：基础设施
  ├─ Nacos (3 副本)
  ├─ MySQL (主从 + Read Replica)
  ├─ Redis (集群)
  ├─ Kafka (3+ Broker)
  ├─ ClickHouse (分片 + 副本)
  ├─ Elasticsearch (3+ Master + Data 节点)
  ├─ MinIO / OSS
  └─ Vault / KMS

阶段 2：监控与日志
  ├─ Prometheus + AlertManager
  ├─ Grafana
  ├─ SkyWalking (OAP + UI)
  ├─ ELK / Loki
  └─ ServiceMonitor 配置

阶段 3：中台服务
  ├─ analytics-gateway
  ├─ analytics-auth
  ├─ analytics-collector
  ├─ analytics-meta
  ├─ analytics-metric
  ├─ analytics-segment
  ├─ analytics-reach
  ├─ analytics-experiment
  ├─ analytics-report
  ├─ analytics-decision
  ├─ analytics-insight
  ├─ analytics-assistant
  ├─ analytics-openapi
  ├─ analytics-admin
  └─ analytics-job

阶段 4：前端
  ├─ analytics-platform-frontend (主管理后台)
  ├─ analytics-portal (开发者门户)
  └─ analytics-docs (文档站)
```

## 四、部署流程

### 首次部署

```bash
# 1. 准备 Helm 仓库
helm repo add yourcompany-analytics https://harbor.yourcompany.com/chartrepo/analytics

# 2. 创建命名空间
kubectl create namespace analytics-prod

# 3. 创建 Secrets（从 Vault 拉取）
kubectl apply -f secrets/ -n analytics-prod

# 4. 部署基础设施（如非托管）
helm install -n analytics-prod nacos yourcompany-analytics/nacos -f values-prod.yaml
helm install -n analytics-prod mysql yourcompany-analytics/mysql -f values-prod.yaml
# ... 其他基础设施

# 5. 等待基础设施 Ready
kubectl wait --for=condition=ready pod -l app=nacos -n analytics-prod --timeout=300s

# 6. 部署中台服务
helm install -n analytics-prod analytics-platform yourcompany-analytics/analytics-platform \
  --version 1.0.0 \
  -f values-prod.yaml \
  --set image.tag=v1.0.0
```

### 滚动升级

```bash
# 通过 Argo CD 自动同步（GitOps）
# 或手动：
helm upgrade -n analytics-prod analytics-platform yourcompany-analytics/analytics-platform \
  --version 1.0.1 \
  --reuse-values \
  --set image.tag=v1.0.1
```

### 灰度发布（Argo Rollouts）

```yaml
# 自动按权重分流：10% → 50% → 100%
# 每阶段间自动分析指标
# 详见 docs/prompts/18-cicd-quality-gate.md
```

### 回滚

```bash
# 一键回滚
kubectl argo rollouts undo analytics-data -n analytics-prod

# 回滚到指定版本
kubectl argo rollouts undo analytics-data --to-revision=5 -n analytics-prod

# 紧急回滚（绕过 Argo Rollouts）
kubectl rollout undo deployment/analytics-data -n analytics-prod
```

## 五、配置管理

### Nacos 命名空间

```
analytics-dev      / 配置组：DEFAULT_GROUP
analytics-test     / 配置组：DEFAULT_GROUP
analytics-staging  / 配置组：DEFAULT_GROUP
analytics-prod     / 配置组：DEFAULT_GROUP
```

### Secret 管理

- **应用密钥**：Vault Transit / Sealed Secrets
- **数据库密码**：Vault Database Secrets Engine（动态密码）
- **Token / JWT Secret**：Vault KV
- **AppSecret**：Vault KV，业务方自助轮换

### 配置文件优先级

```
1. 命令行参数（最高）
2. 系统环境变量
3. application-{env}.yml
4. Nacos 远程配置
5. application.yml（默认）
6. 代码默认值（最低）
```

## 六、HPA 与资源规划

### 资源配置示例

```yaml
# analytics-collector
resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 4000m
    memory: 8Gi

# analytics-assistant (LLM 应用)
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi

# analytics-gateway
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
```

### HPA 策略

| 服务 | 触发指标 | 阈值 | 副本范围 |
|---|---|---|---|
| collector | CPU + QPS | 70% / 8000 | 3-50 |
| gateway | CPU | 70% | 3-20 |
| metric | CPU | 70% | 2-10 |
| segment | CPU | 70% | 2-10 |
| reach | CPU | 70% | 2-10 |
| assistant | LLM 队列长度 | 100 | 2-20 |
| insight | 定时任务 | — | 2 固定 |

### PDB（Pod Disruption Budget）

所有服务最小可用：`minAvailable: 50%`

## 七、网络与流量

### 入口

```
Internet
   ↓
Cloudflare / 阿里云 WAF（DDoS 防护）
   ↓
Ingress Controller (Nginx / Traefik)
   ↓
analytics-gateway
   ↓
各微服务
```

### 内部通信

- Service Mesh（可选）：Istio / Linkerd
- 默认：K8s Service ClusterIP + Spring Cloud LoadBalancer

### 出口

- 调用 LLM API：通过 Egress Gateway 统一审计
- 调用第三方服务：白名单 IP

## 八、数据存储部署

### MySQL

- 主从（1 主 + 2 从）
- 主：写
- 从：读 + 备份
- 每日备份至 OSS
- 增量备份（binlog）

### ClickHouse

- 分片 ×3 + 副本 ×2 = 6 节点
- ZooKeeper 集群（3 节点）
- 数据按 `cityHash64(user_id)` 分布
- 冷热分层：30 天热（SSD）→ 90 天温（HDD）→ 归档（OSS）

### Redis

- 集群模式（3 主 + 3 从）
- 持久化：AOF + RDB
- 主从切换：Redis Sentinel 或托管

### Kafka

- 3+ Broker
- 副本数 ≥ 3
- 分区数按吞吐预估
- 数据保留 7 天

## 九、首次上线 Checklist

- [ ] 基础设施全部就绪并通过健康检查
- [ ] Vault 密钥配置完成
- [ ] Nacos 配置完成（各环境隔离）
- [ ] 数据库 Schema 已迁移（Flyway）
- [ ] 监控告警已配置
- [ ] DNS 已配置（含 SSL 证书）
- [ ] WAF 规则已配置
- [ ] OneID 映射数据已导入
- [ ] 至少 1 个 P0 业务方接入沙箱验证
- [ ] 灰度发布配置已就绪（Argo Rollouts）
- [ ] 回滚预案已演练
- [ ] 运维 oncall 排班已确定

## 十、运维 oncall 制度

- **L1 oncall**：值班工程师，7×24，10 分钟响应
- **L2 oncall**：模块 Owner，工作日 +夜间紧急，30 分钟响应
- **L3 oncall**：架构师 / 总监，重大故障升级

排班工具：PagerDuty / 自建

## 十一、附录

- 详细 Helm values 见 `helm/analytics-platform/values-{env}.yaml`
- 详细监控告警规则见 [02-monitoring-alerting.md](./02-monitoring-alerting.md)
- 故障排查指南见 [03-troubleshooting.md](./03-troubleshooting.md)
- 灾备方案见 [05-disaster-recovery.md](./05-disaster-recovery.md)
