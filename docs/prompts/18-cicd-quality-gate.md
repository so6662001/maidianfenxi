# 提示词 18 · CI/CD 与质量门禁

> 前置：`00-master-prompt.md`。

---

## 提示词正文

```
你是 DevOps 工程师。请为 analytics-platform 搭建完整 CI/CD 流水线 + 质量门禁。

【目标】
代码提交到生产部署的全自动化流水线，含质量门禁、安全扫描、灰度发布、监控告警。

【技术】

- GitLab CI（推荐）或 Jenkins / GitHub Actions
- Docker multi-stage build
- Helm 3 + Argo CD（GitOps）
- Harbor 镜像仓库
- SonarQube（代码质量）
- Trivy / Snyk（安全扫描）
- Argo Rollouts（蓝绿 / 金丝雀）

【流水线阶段】

```
.gitlab-ci.yml

stages:
  - lint           # 代码格式 + 静态检查
  - unit-test      # 单元测试 + 覆盖率
  - security-scan  # 依赖漏洞 + SAST
  - build          # 编译打包
  - image          # 构建镜像 + 推送
  - deploy-dev     # 自动部署 dev
  - integration-test # 集成测试
  - deploy-staging # 自动部署 staging
  - e2e-test       # E2E 测试
  - manual-approve # 人工审批
  - deploy-prod    # 部署 prod（灰度）
```

【质量门禁（必须通过才能进下一阶段）】

| 阶段 | 门禁 |
|---|---|
| lint | CheckStyle 0 错误，Spotless 通过 |
| unit-test | 覆盖率 ≥ 60%，核心模块 ≥ 80%，无失败用例 |
| security-scan | 无 Critical / High 漏洞 |
| sonar | 重复率 < 5%，无 Blocker / Critical issue |
| build | mvn package 通过，无警告（除允许列表） |
| image | 镜像扫描无 Critical 漏洞，体积 < 250MB |
| e2e | 100% 通过 |

【.gitlab-ci.yml 示例】

```yaml
default:
  image: maven:3.9-eclipse-temurin-17
  cache:
    paths:
      - .m2/repository

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

lint:
  stage: lint
  script:
    - mvn spotless:check checkstyle:check
  only:
    - merge_requests
    - main

unit-test:
  stage: unit-test
  script:
    - mvn test jacoco:report
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: '**/target/surefire-reports/TEST-*.xml'
      coverage_report:
        coverage_format: jacoco
        path: '**/target/site/jacoco/jacoco.xml'

security-scan:
  stage: security-scan
  script:
    - mvn org.owasp:dependency-check-maven:check
    - snyk test --severity-threshold=high
  allow_failure: false

sonar:
  stage: security-scan
  script:
    - mvn sonar:sonar -Dsonar.host.url=$SONAR_URL -Dsonar.login=$SONAR_TOKEN
  only:
    - merge_requests
    - main

build:
  stage: build
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - '**/target/*.jar'

image:
  stage: image
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $HARBOR_USER -p $HARBOR_PASS $HARBOR_URL
    - for module in analytics-gateway analytics-data analytics-app analytics-assistant; do
        docker build -t $HARBOR_URL/analytics/$module:$CI_COMMIT_SHA -f $module/Dockerfile $module
        trivy image --severity CRITICAL --exit-code 1 $HARBOR_URL/analytics/$module:$CI_COMMIT_SHA
        docker push $HARBOR_URL/analytics/$module:$CI_COMMIT_SHA
      done

deploy-dev:
  stage: deploy-dev
  image: alpine/helm:3.12
  script:
    - helm upgrade --install analytics-platform helm/analytics-platform/
        --namespace analytics-dev
        --set image.tag=$CI_COMMIT_SHA
        --set environment=dev
  only:
    - main
    - develop

deploy-staging:
  stage: deploy-staging
  script: |
    helm upgrade --install ... --namespace analytics-staging
  when: on_success
  only:
    - main

e2e-test:
  stage: e2e-test
  script:
    - npx playwright test --reporter=junit
  artifacts:
    reports:
      junit: e2e-results.xml

manual-approve:
  stage: manual-approve
  script: echo "Approved"
  when: manual
  only:
    - main

deploy-prod:
  stage: deploy-prod
  script: |
    # Argo Rollouts 金丝雀
    kubectl argo rollouts set image analytics-platform \
      analytics-data=$HARBOR_URL/analytics/analytics-data:$CI_COMMIT_SHA \
      -n analytics-prod
    kubectl argo rollouts promote analytics-platform -n analytics-prod
  when: on_success
  only:
    - main
```

【多阶段 Dockerfile】

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY analytics-common/pom.xml analytics-common/
COPY analytics-data/pom.xml analytics-data/
RUN mvn dependency:go-offline -B
COPY . .
RUN mvn package -pl analytics-data -am -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY --from=builder /app/analytics-data/target/*.jar app.jar

# 非 root 用户
RUN useradd -r -u 1000 -g root appuser
USER appuser

EXPOSE 8001
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8001/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseG1GC", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Dfile.encoding=UTF-8", \
  "-Duser.timezone=Asia/Shanghai", \
  "-javaagent:/opt/skywalking/skywalking-agent.jar", \
  "-jar", "/app/app.jar"]
```

【Helm Chart 结构】

```
helm/analytics-platform/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── networkpolicy.yaml
│   ├── servicemonitor.yaml   # Prometheus
│   └── _helpers.tpl
```

values.yaml 必须含：
- 资源 requests/limits（显式声明）
- HPA 配置（CPU + QPS）
- PDB（保证 N-1 可用）
- Nacos / Redis / Kafka / MySQL / ClickHouse 配置
- Prometheus ServiceMonitor

【Argo Rollouts 金丝雀】

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: { duration: 10m }
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 100
```

【AnalysisTemplate 示例】

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{status=~"2.."}[5m]))
          / sum(rate(http_requests_total[5m]))
```

【Prometheus 告警规则】

```yaml
groups:
- name: analytics-platform
  rules:
  - alert: HighErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
    for: 1m
    severity: critical
  - alert: HighLatency
    expr: histogram_quantile(0.95, http_request_duration_seconds) > 1
    for: 5m
    severity: warning
  - alert: KafkaLag
    expr: kafka_consumer_lag > 100000
    for: 5m
    severity: warning
  - alert: PodCrashLoop
    expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
    severity: critical
```

【监控告警通道】

- 飞书机器人 / 钉钉机器人 / 企微机器人
- 邮件（重要）
- SMS（仅 P0）
- Alertmanager 路由配置

【日志聚合】

- Filebeat → Kafka → Logstash → Elasticsearch → Kibana
- 或 Loki + Promtail（轻量）

【一键回滚】

```bash
# 任意历史版本回滚
kubectl argo rollouts undo analytics-platform -n analytics-prod
# 或指定版本
kubectl argo rollouts undo analytics-platform --to-revision=5 -n analytics-prod
```

【验证】

1. 一次提交全流程跑通
2. 任一门禁失败正确阻断
3. 金丝雀发布按比例分流
4. 监控告警触达
5. 回滚 < 2min

【交付物】

- 完整 .gitlab-ci.yml
- 各服务 Dockerfile
- 完整 Helm Charts
- Argo Rollouts 配置
- Prometheus 告警规则
- Alertmanager 配置
- 部署文档 / 运维手册 / 故障排查手册

【禁止】

- 跳过门禁强制部署
- 镜像 root 用户
- 硬编码密钥
- 资源不设 limits

【安全】

- 镜像必须 distroless 或 jre-jammy
- 非 root 用户
- 密钥走 Vault / Sealed Secrets
- 镜像扫描门禁
- 依赖扫描门禁

现在请生成完整 CI/CD 配置。
```
