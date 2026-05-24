# 测试策略总览

## 一、测试金字塔

```
       ┌─────────────────┐
       │  E2E（10%）      │  关键业务路径
       ├─────────────────┤
       │  集成（30%）     │  服务间联调
       ├─────────────────┤
       │  单元（60%）     │  函数/类级
       └─────────────────┘
```

## 二、覆盖率门禁

| 类型 | 整体 | 核心模块 |
|---|---|---|
| 单元测试 | ≥ 60% | ≥ 80% |
| 集成测试 | 核心 API 100% | — |
| E2E | 关键路径 100% | — |
| 性能测试 | 每次发版 | — |
| 安全测试 | 每次 PR + 季度渗透 | — |
| 兼容性测试 | 主流浏览器 + Java 版本 | — |

**核心模块**：鉴权、指标计算、决策追溯、AI 助理 Function 调用、数据采集。

## 三、单元测试

### 后端（Java + JUnit 5）

```java
@SpringBootTest
class MetricServiceTest {
    @MockBean ClickHouseClient ckClient;
    @Autowired MetricService metricService;
    
    @Test
    @DisplayName("应该正确路由到物化视图")
    void shouldRouteToMaterializedView() {
        // Given
        MetricQuery query = MetricQuery.of("M-SE-001", "day");
        when(ckClient.execute(any())).thenReturn(mockResult);
        
        // When
        MetricResult result = metricService.query(query);
        
        // Then
        assertThat(result.getValue()).isEqualTo(1.12);
        verify(ckClient).execute(argThat(sql -> sql.contains("dws_nrr_monthly")));
    }
    
    @Test
    @DisplayName("超时应该 kill 查询")
    void shouldKillQueryOnTimeout() { ... }
    
    @Test
    @DisplayName("权限隔离测试")
    void shouldEnforceRowLevelPermission() { ... }
}
```

**规范**：
- 命名：`should{Behavior}_when{Condition}`
- AAA 模式（Arrange-Act-Assert）
- 一个测试只测一个行为
- 测试数据用 `@TestData` 或 builder
- Mock 用 Mockito / MockBean
- 测试夹具用 @TestConfiguration

### 前端（Vitest + Vue Test Utils）

```ts
import { mount } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import KpiCard from '@/shared/components/KpiCard.vue';

describe('KpiCard', () => {
  it('renders title and value correctly', () => {
    const wrapper = mount(KpiCard, {
      props: { title: '总营收', value: '¥4.2亿', trend: 'up', trendValue: '+12%' }
    });
    expect(wrapper.text()).toContain('总营收');
    expect(wrapper.text()).toContain('¥4.2亿');
  });
  
  it('shows warning icon when status is warning', () => {
    const wrapper = mount(KpiCard, { props: { ..., status: 'warning' } });
    expect(wrapper.find('.warning-icon').exists()).toBe(true);
  });
  
  it('emits click event', async () => {
    const wrapper = mount(KpiCard, { props: { ..., onClick: () => {} } });
    await wrapper.trigger('click');
    expect(wrapper.emitted('click')).toBeTruthy();
  });
});
```

## 四、集成测试

### 后端集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class MetricControllerIT {
    @Container static MySQLContainer mysql = new MySQLContainer("mysql:8.0");
    @Container static GenericContainer<?> ck = new GenericContainer<>("clickhouse/clickhouse-server:23");
    @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7");
    
    @Autowired MockMvc mockMvc;
    
    @Test
    void shouldQueryMetricSuccessfully() throws Exception {
        mockMvc.perform(post("/api/metric/query")
                .header("Authorization", "Bearer " + jwt)
                .contentType(APPLICATION_JSON)
                .content("""{"metric":"M-SE-001","granularity":"day"}"""))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(0))
            .andExpect(jsonPath("$.data.rows").isArray());
    }
}
```

### 契约测试（Pact / Spring Cloud Contract）

业务方与中台之间的接口契约：

```groovy
// Provider side
Contract.make {
    description "should return user profile"
    request {
        method GET()
        url "/openapi/v1/user/profile/u-001"
        headers { ... }
    }
    response {
        status 200
        body([
            code: 0,
            data: [
                basic: [name: "Alice", email: "..."]
            ]
        ])
    }
}
```

## 五、E2E 测试（Playwright）

### 关键路径必测

| 路径 | 优先级 |
|---|---|
| 登录 → S-1 看板 → 查看产品详情 → 记录决策 | P0 |
| 业务方 SDK 上报 → Collector → ClickHouse → 看板查询 | P0 |
| 创建分群 → 配置 Journey → 启动触达 | P0 |
| 决策助理对话 → 数据查询 → 一键圈人 | P0 |
| AB 实验创建 → 运行 → 查看结果 → 归档 | P1 |
| 嵌入式 UserProfile360 在业务系统使用 | P1 |
| Playbook 启动 → 实验 → 复盘 | P1 |

### Playwright 示例

```ts
import { test, expect } from '@playwright/test';

test('Total PM can record decision from S-1', async ({ page }) => {
  // 登录
  await page.goto('/login');
  await page.fill('#username', 'totalpm');
  await page.fill('#password', 'xxx');
  await page.click('button[type=submit]');
  
  // 进入 S-1
  await page.click('text=S-1 产品组合矩阵');
  await expect(page).toHaveURL(/.*\/s1/);
  
  // 等待数据加载
  await page.waitForSelector('.kpi-card', { state: 'visible' });
  
  // 点击 AI 建议
  await page.click('text=记录决策');
  
  // 填写决策表单
  await page.fill('#title', '测试决策');
  await page.selectOption('#type', 'product');
  await page.fill('#chosen-option', 'A');
  await page.fill('#rationale', '测试理由');
  
  // 提交
  await page.click('text=提交');
  
  // 验证成功
  await expect(page.locator('.success-toast')).toContainText('决策已登记');
});
```

### E2E 测试运行

```bash
pnpm test:e2e             # 全部
pnpm test:e2e --grep S-1  # 单条
pnpm test:e2e:debug       # debug 模式
pnpm test:e2e:ui          # UI 模式
```

## 六、性能测试

### 工具
- 后端：JMeter / wrk / Gatling / k6
- 前端：Lighthouse / WebPageTest
- 数据库：sysbench / clickhouse-benchmark

### 后端压测场景

```javascript
// k6 示例
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },
    { duration: '5m', target: 1000 },
    { duration: '2m', target: 5000 },
    { duration: '5m', target: 5000 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.post('https://api.../openapi/v1/metric/query', JSON.stringify({...}), {
    headers: { 'X-App-Key': 'xxx', /* signature... */ }
  });
  check(res, { 'status 200': r => r.status === 200 });
}
```

### 性能目标（详见 SLO 文档）

每次发版前性能测试必须达标。

### 前端性能

- Lighthouse 性能 ≥ 90
- LCP < 2.5s
- FID < 100ms
- CLS < 0.1
- 首屏 < 2s
- TTI < 3.8s

## 七、安全测试

### 1. SAST（静态扫描）

```yaml
# CI 集成
- SonarQube：代码质量 + 安全 issue
- Checkmarx / Snyk Code：深度静态扫描
- 阿里 P3C：Java 编码规约
```

### 2. SCA（依赖扫描）

```yaml
- OWASP Dependency Check
- Snyk
- Trivy
```

每次 PR 触发，无 Critical / High 漏洞才能合并。

### 3. DAST（动态扫描）

```yaml
- OWASP ZAP：发版前自动扫描
- Nessus / Burp Suite：人工渗透
```

### 4. 镜像扫描

```yaml
- Trivy：构建镜像后立即扫描
- Harbor 内置 Clair
```

### 5. 渗透测试

- 季度内部渗透
- 半年外部团队渗透
- 输出报告 + 修复跟踪

### 6. 红蓝对抗

- 年度一次
- 模拟真实攻击场景

### 7. 合规审计

- 年度 PIPL / GDPR / 数据安全法合规审计
- 输出合规报告

## 八、兼容性测试

### 浏览器
- Chrome 最新 2 版
- Edge 最新 2 版
- Safari 最新 2 版
- Firefox 最新 2 版

### Java 版本（SDK）
- Java 8 / 11 / 17 / 21

### Vue 版本（前端 SDK）
- Vue 3.4+（peer dependency）

### 移动端（响应式）
- iOS Safari
- Chrome Android

## 九、混沌工程

详见 [03-chaos-engineering.md](./03-chaos-engineering.md)。

## 十、AI 助理评估测试集

### 30 场景自动评估

```ts
// tests/assistant-evaluation.ts
const testCases = [
  {
    scenario: 'Q-001 早间汇报',
    input: '今天有什么需要决策的？',
    expected: {
      hasData: true,
      hasExplanation: true,
      hasActions: true,
      maxLatencyMs: 5000,
      maxLlmCost: 0.5
    }
  },
  // ... 30 个场景
];

// 每次助理服务发版必跑
```

### 评估指标

- 准确率（人工抽样）≥ 80%
- 响应时间 P95 < 5s
- 用户 👍 率 ≥ 70%
- 工具调用成功率 ≥ 99%
- LLM 幻觉率 ≤ 5%

## 十一、回归测试

每次发版必跑：
- 单元测试（CI 自动）
- 集成测试（CI 自动）
- E2E 关键路径（CI 自动）
- 性能基准（CI 自动）

每周跑：
- 完整 E2E
- 兼容性测试

## 十二、测试数据管理

### 测试数据集

- `test-data/users.json` 100 测试用户
- `test-data/events.json` 1000 测试事件
- `test-data/segments.json` 标准分群
- `test-data/decisions.json` 决策案例

### 数据脱敏

生产数据导入测试环境前必须脱敏：
- 手机号 → 138****8888
- 姓名 → 张** / Alice*
- 邮箱 → x***@example.com
- 订单金额 → 模糊化

### 测试数据生命周期

- dev：每周清理重置
- test：每月清理重置
- staging：保留 1 月
- E2E：每次跑前重置

## 十三、测试 CI 集成

```yaml
# .gitlab-ci.yml
stages:
  - unit-test
  - integration-test
  - security-scan
  - e2e
  - performance

unit-test:
  script: mvn test
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: '**/target/surefire-reports/TEST-*.xml'
      coverage_report:
        coverage_format: jacoco
        path: '**/target/site/jacoco/jacoco.xml'

# ... 详见 docs/prompts/18-cicd-quality-gate.md
```

## 十四、测试治理

- 每周 1 次测试用例评审
- 每月统计：覆盖率 / 通过率 / 缺陷分布
- 季度：测试策略评估 + 工具升级
- 年度：测试基础设施升级
