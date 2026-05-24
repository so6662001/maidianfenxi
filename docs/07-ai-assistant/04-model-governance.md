# AI 模型治理

## 一、为什么需要 AI 模型治理

LLM 不像传统软件那样确定性运行，存在：
- **效果不确定**：同一 Prompt 不同时间结果可能不同
- **成本不可控**：Token 消耗可能突增
- **质量回归**：模型升级后某些场景可能变差
- **多 Provider 风险**：OpenAI / Anthropic / 国内厂商 各有不同
- **合规风险**：数据出境、内容审核、版权

需要系统化治理。

## 二、模型清单管理

### 已使用模型登记

```yaml
models:
  - id: gpt-4-turbo
    provider: openai
    type: large
    cost_per_1k_input: 0.01 USD
    cost_per_1k_output: 0.03 USD
    use_cases: [复杂推理, 决策辅助]
    status: active
    fallback: claude-3-opus
    
  - id: gpt-4o-mini
    provider: openai
    type: small
    cost_per_1k_input: 0.00015 USD
    cost_per_1k_output: 0.0006 USD
    use_cases: [意图识别, 简单查询]
    status: active
    
  - id: claude-3-opus
    provider: anthropic
    type: large
    cost: ...
    use_cases: [长文本, 复杂分析]
    status: active
    
  - id: qwen-max
    provider: alibaba
    type: large
    cost: ...
    use_cases: [国内合规场景]
    status: active
    
  - id: deepseek-v3
    provider: deepseek
    type: large
    cost: ...
    use_cases: [代码生成, 国产替代]
    status: testing
```

### 模型生命周期

```
testing（小流量灰度）
   ↓ 验证
active（生产使用）
   ↓ 评估
deprecated（推荐迁移）
   ↓ 等待
removed（下线）
```

## 三、Prompt 版本管理

### Prompt 即代码

所有 Prompt 必须：
- 存 Git（带版本号）
- 评审（PR + Code Review）
- 测试（评估测试集）
- 灰度发布
- 监控效果

### Prompt 仓库结构

```
prompts/
├── assistant/
│   ├── system-prompt-v1.0.md
│   ├── system-prompt-v1.1.md
│   ├── system-prompt-v2.0.md     # 当前 active
│   └── CHANGELOG.md
├── insight/
│   ├── anomaly-attribution-v1.0.md
│   └── decision-suggestion-v1.0.md
├── decision/
│   └── review-draft-v1.0.md
└── tools/
    └── function-calling-prompts/
```

### Prompt 元数据

```yaml
prompt_id: assistant-system-v2.0
description: 决策助理主系统提示词
author: AI PM
created: 2026-05-01
status: active
test_results:
  - test_set: 30-scenarios-v1
    pass_rate: 0.92
    avg_latency: 2.3s
    avg_cost: ¥0.05
deployment:
  - environment: prod
    deployed_at: 2026-05-15
    traffic_pct: 100
ab_test_history:
  - vs: v1.0
    result: v2.0 提升采纳率 8pp，无显著回归
```

### Prompt 变更流程

```
1. 提出变更（PR）
   ↓
2. 评审（AI PM + 资深 Prompt 工程师）
   ↓
3. 回归测试（30 场景评估测试集）
   ↓
4. 灰度发布（10% → 50% → 100%）
   ↓
5. 监控（采纳率 / 满意度 / 成本）
   ↓
6. 全量或回滚
   ↓
7. 归档变更记录到决策追溯库
```

## 四、模型评估测试集

### 标准评估集

针对每个核心场景准备 ≥ 30 个标准测试用例：

```ts
const evaluationSet = [
  {
    id: 'EVAL-001',
    scenario: 'L1-数据查询',
    input: '上周 SaaS DAU 是多少？',
    expected: {
      hasData: true,
      hasMetric: 'DAU',
      hasTimeRange: 'last_week',
      hasActions: true,
      maxLatencyMs: 3000,
      maxCost: 0.05,
      noPII: true,
    }
  },
  {
    id: 'EVAL-002',
    scenario: 'L3-归因',
    input: '为什么平台询盘下降？',
    expected: {
      hasBreakdown: true,
      hasRelatedEvents: true,
      hasSuggestions: true,
      suggestionsCount: { min: 1, max: 3 },
    }
  },
  // ... 30+ cases
];
```

### 自动评估指标

| 指标 | 计算 | 目标 |
|---|---|---|
| 一次性命中率 | passed / total | ≥ 60% |
| 平均响应时长 | avg(latency) | ≤ 5s |
| 平均 Token 消耗 | avg(tokens) | 按场景 |
| 平均成本 | avg(cost) | 按场景 |
| 工具调用成功率 | tool_success / total_tool_call | ≥ 99% |
| 越权数据访问 | 必须 0 | 0 |
| PII 泄露 | 必须 0 | 0 |

### 人工评估

每月抽样 100 个真实对话，人工评估：
- 准确率
- 有用性
- 安全性
- 体验

## 五、模型监控

### 业务指标

```yaml
# Prometheus 指标
assistant_chat_total{model, scene}
assistant_chat_thumbsup_total{model, scene}
assistant_chat_thumbsdown_total{model, scene}
assistant_chat_followup_total{model, scene}  # 用户继续追问
assistant_chat_abandoned_total{model, scene}
```

### 性能指标

```yaml
assistant_llm_latency_seconds{model, step}
assistant_llm_tokens_total{model, direction}
assistant_llm_cost_yuan{model, scene}
assistant_llm_error_total{model, error_type}
```

### 安全指标

```yaml
assistant_prompt_injection_detected_total
assistant_permission_denied_total
assistant_pii_leak_total
assistant_content_audit_blocked_total
```

## 六、模型对比与切换

### A/B 实验框架

```yaml
experiment:
  name: gpt-4o-mini-vs-qwen-turbo
  hypothesis: Qwen-Turbo 在意图识别场景成本降低 50%，质量不降
  variants:
    control: gpt-4o-mini (50%)
    treatment: qwen-turbo (50%)
  metrics:
    primary: intent_classification_accuracy
    secondary: [latency, cost]
    guardrail: [satisfaction]
  duration: 14d
```

### 切换决策

```
新模型在 evaluation 测试集上 ≥ 旧模型
   ↓
小流量 A/B（5%）
   ↓
监控 1-2 周
   ↓
通过：扩到 50% → 100%
不通过：回滚 + 复盘
```

## 七、模型成本管理

### 成本预算

```yaml
budget:
  monthly_total: ¥500,000
  by_model:
    gpt-4: ¥200,000
    claude-3: ¥150,000
    qwen-max: ¥100,000
    others: ¥50,000
  by_scene:
    decision_assistant: ¥300,000
    intelligent_insight: ¥100,000
    decision_review: ¥50,000
    others: ¥50,000
```

### 成本告警

- 月预算 50% → 通知
- 月预算 80% → 警告
- 月预算 100% → 限流（降级到小模型）

### 成本优化策略

1. **模型路由**：详见 P-AU-001
2. **语义缓存**：相似请求复用结果
3. **输出长度控制**：默认精简
4. **批量处理**：合并同类请求
5. **Prompt 优化**：减少 Token

## 八、安全治理

### Prompt 注入防护

```java
// 1. 系统 Prompt 与用户输入严格分离
String userInput = sanitize(rawInput);  // 清理
String prompt = systemPrompt + "\n\n[USER INPUT START]\n" + userInput + "\n[USER INPUT END]";

// 2. 检测注入尝试
if (containsInjectionPattern(userInput)) {
    log.warn("Prompt injection detected: user={}", userId);
    return rejectResponse();
}

// 3. 输出过滤
if (outputContainsSystemInfo(response)) {
    log.warn("Output leak detected");
    return sanitizedResponse;
}
```

### 内容审核

- 输入：敏感词过滤 + 政治内容检测
- 输出：模型自带审核 + 二次过滤
- 不合规内容拒绝输出 + 记录

### 数据隐私

- 用户对话不参与模型训练（明确告知）
- 对话日志脱敏存储
- 不传敏感数据给第三方 LLM（用脱敏后摘要）

## 九、模型迭代日志

每次模型/Prompt 变更登记到决策追溯库：

```yaml
变更 ID: AI-MODEL-2026-0524-001
变更类型: Prompt 升级 / 模型切换 / 路由策略调整
变更内容: System Prompt v1.0 → v2.0
评估结果:
  - 采纳率：+8pp
  - 成本：-15%
  - 满意度：+5pp
风险评估:
  - 影响范围：所有助理对话
  - 回退方案：保留 v1.0，1 键回滚
上线策略: 灰度 10% → 50% → 100%（每阶段 3 天）
责任人: AI PM 张三
```

## 十、第三方依赖管理

### 多 Provider 容错

```yaml
provider_strategy:
  primary: openai
  fallback_chain:
    - anthropic
    - alibaba_qwen
    - baidu_ernie
  
  routing_rules:
    - if: provider_unavailable
      then: switch_to_fallback
    - if: rate_limited
      then: queue_then_retry
    - if: latency_high
      then: switch_to_smaller_model
```

### 供应商评估

季度评估：
- 价格变化
- 模型能力（新模型评估）
- 服务稳定性
- 合规情况

## 十一、合规

### 数据出境

- 使用国外 LLM（OpenAI / Claude）→ 数据可能出境
- 必须做"数据出境安全评估"
- 涉敏数据用国内模型（通义千问 / 文心一言）
- 用户可选偏好

### 国内合规

- 通过《生成式人工智能服务管理暂行办法》备案
- 内容审核机制
- 不生成违法违规内容

## 十二、文档与培训

- AI 模型治理手册（本文）
- Prompt 工程培训（季度）
- 安全意识培训（半年）
- 案例库（决策追溯库 AI 案例）

## 十三、AI 治理 KPI

| 指标 | 目标 |
|---|---|
| Prompt 变更评审完成率 | 100% |
| 评估测试集通过率 | ≥ 90% |
| 模型成本预算执行 | ±10% |
| 越权事件 | 0 |
| 内容合规事件 | 0 |
| 模型切换成功率 | ≥ 95% |
| 月度模型评估完成率 | 100% |
