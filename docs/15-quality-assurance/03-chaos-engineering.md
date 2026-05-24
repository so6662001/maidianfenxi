# 混沌工程方案

## 一、为什么需要混沌工程

> "你不能从未发生的故障中学习"——Werner Vogels

通过**主动注入故障**，在受控环境中发现系统弱点，提升韧性。

## 二、混沌工程原则

1. **从生产环境的稳态行为出发**（建立假设）
2. **改变现实世界的事件**（注入故障）
3. **在生产中进行实验**（影响可控）
4. **持续自动化**
5. **最小化爆炸半径**

## 三、工具选型

| 工具 | 用途 |
|---|---|
| **ChaosMesh** | K8s 原生，故障类型丰富 |
| **Chaos Monkey** | Netflix 经典，简单 |
| **Litmus** | K8s + 多种场景 |
| **Gremlin** | 商业，功能全 |
| **PowerfulSeal** | K8s 故障 |

**推荐**：ChaosMesh（开源 + K8s 集成）

## 四、故障注入清单

### 基础设施层

| 故障 | 影响 | 验证目标 |
|---|---|---|
| Pod Kill | 单 Pod 终止 | K8s 自动重启 / 业务无感 |
| Pod Failure | Pod 不响应 | 服务发现剔除 / 切流 |
| Container CPU 100% | 资源耗尽 | HPA 扩容 / 限流 |
| Container Memory OOM | 内存溢出 | 自动重启 / 监控告警 |
| Node Down | 单节点故障 | K8s 调度迁移 |
| Network Delay | 网络延迟 | 超时机制 / 重试 |
| Network Packet Loss | 丢包 | 重试 / 降级 |
| Network Partition | 网络分区 | 脑裂处理 |
| DNS Failure | DNS 解析失败 | 服务发现兜底 |

### 中间件层

| 故障 | 验证目标 |
|---|---|
| MySQL 主节点宕机 | 自动 failover / 数据一致性 |
| Redis 集群部分节点故障 | 主从切换 / 缓存降级 |
| Kafka Broker 故障 | 副本选主 / Producer 缓冲 |
| ClickHouse 副本故障 | 查询自动路由 |
| Elasticsearch 节点故障 | 副本接管 |

### 应用层

| 故障 | 验证目标 |
|---|---|
| Service 接口延迟 | 上游超时 / 降级 |
| Service 接口报错 | 上游重试 / 熔断 |
| 配置错误 | 启动失败检测 |
| 日志爆炸 | 日志限流 / 磁盘保护 |

### 外部依赖

| 故障 | 验证目标 |
|---|---|
| LLM API 超时 | 模型切换 / 降级 |
| LLM API 限流 | 重试 / 队列 |
| 邮件 SMTP 不可达 | 备用通道 |
| 短信服务故障 | 重试 / 备用 |
| 第三方 API 故障 | 熔断 / 降级 |

## 五、ChaosMesh 实验示例

### Pod Kill

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-collector-pod
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - analytics-staging
    labelSelectors:
      app: analytics-collector
  scheduler:
    cron: "0 14 * * 1"  # 每周一 14:00
```

### 网络延迟

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: delay-metric-to-clickhouse
spec:
  action: delay
  mode: all
  selector:
    labelSelectors:
      app: analytics-metric
  delay:
    latency: "500ms"
    correlation: "25"
    jitter: "100ms"
  duration: "10m"
  direction: to
  target:
    selector:
      labelSelectors:
        app: clickhouse
    mode: all
```

### CPU 压力

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-assistant
spec:
  selector:
    labelSelectors:
      app: analytics-assistant
  mode: one
  stressors:
    cpu:
      workers: 4
      load: 90
  duration: "5m"
```

### 数据库故障

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-mysql-primary
spec:
  action: pod-kill
  mode: one
  selector:
    labelSelectors:
      app: mysql
      role: primary
```

## 六、混沌实验流程

```
1. 选定假设（如：collector 单 Pod 故障，业务无感知）
   ↓
2. 评估爆炸半径（在 staging 还是 prod？影响多少用户？）
   ↓
3. 制定回滚方案（紧急停止实验）
   ↓
4. 准备监控（重点观察哪些指标）
   ↓
5. 执行实验（ChaosMesh 注入）
   ↓
6. 观察 + 验证假设
   ↓
7. 停止实验（自动或手动）
   ↓
8. 复盘 + 改进
   ↓
9. 归档（决策追溯库）
   ↓
10. 自动化 + 周期执行
```

## 七、混沌实验日历

### 每周（staging 环境）
- 周一 14:00：单 Pod Kill
- 周三 14:00：网络延迟注入
- 周五 14:00：CPU 压力

### 每月（staging 环境）
- 模拟主节点故障
- 模拟 AZ 故障
- 全链路压测

### 每季度（生产环境，受控）
- 模拟单 Region 故障（仅灰度流量）
- 模拟核心依赖故障
- 完整 DR 演练

## 八、爆炸半径控制

### 原则
- staging 优先（先在非生产环境验证）
- 流量限制（生产只对 10% 流量做混沌）
- 时间限制（实验最长 30 min）
- 紧急停止（5 min 内可停止）
- 业务峰值时段避开

### 紧急停止

```bash
# ChaosMesh 紧急停止所有实验
kubectl delete podchaos --all -n chaos-testing
kubectl delete networkchaos --all -n chaos-testing
```

## 九、稳态指标

实验前定义稳态指标，监控偏离：

| 指标 | 稳态阈值 |
|---|---|
| 服务可用性 | ≥ 99.9% |
| API P95 延迟 | < 500ms |
| 业务 SLO | 不破坏 |
| 用户投诉 | 0 |

任一指标恶化 → 立即停止实验。

## 十、混沌实验复盘模板

```yaml
实验编号: CHAOS-2026-0524-001
实验名称: collector Pod Kill
执行时间: 2026-05-24 14:00 - 14:15
环境: staging
假设: collector 单 Pod 故障，业务调用无感
实际结果:
  - K8s 自动重启 Pod（30s）
  - 期间错误率：0.05%（流量切到其他 Pod）
  - 业务无感知 ✓
  
发现问题:
  - 重启期间监控告警 5 min 才触发（应该更快）
  - 某些客户端没有重试机制，导致单次请求失败
  
改进措施:
  - 监控告警阈值调整为 1 min
  - SDK 增加自动重试（已修复）
  - 通知业务方升级 SDK
  
经验沉淀:
  - 知识标签：[Pod故障] [自动恢复] [SDK重试]
  - 自动归入混沌实验案例库
```

## 十一、混沌工程文化

- **失败可被接受**：混沌不是惩罚
- **持续学习**：每次实验都有收获
- **跨团队协作**：开发 + 运维 + SRE 共同参与
- **自动化**：减少人工，避免遗漏
- **知识分享**：实验结果在团队分享

## 十二、上线节奏

### Phase 1（MVP）
- 仅 staging 环境
- 手动触发
- 单一故障类型（Pod Kill / 网络延迟）

### Phase 2（V1）
- 自动化（每周计划）
- 多种故障类型
- 监控对接

### Phase 3（V2）
- 生产环境（受控）
- 复杂场景组合
- ChaosMesh Workflow

### Phase 4（V3）
- AI 驱动的混沌（智能选择实验）
- 持续混沌（Always-on）

## 十三、与其他模块联动

- 实验失败 → 自动登记决策追溯库
- 实验发现的改进项 → 入产品待办池
- 季度评审：所有混沌实验复盘 → 评估系统韧性

## 十四、不在 MVP 范围

- 生产环境混沌（V2 才考虑）
- 跨 Region 混沌（V3）

但**预案必须现在做**（详见灾备方案）。
