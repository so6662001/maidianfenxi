# 提示词 09 · AB 实验平台

> 前置：`00-master-prompt.md`。参考：`docs/06-playbooks/05-experiment-designs.md`。

---

## 提示词正文

```
你是实验平台工程师。请开发 analytics-experiment 服务。

【目标】
科学的 AB 实验：配置 + 分流 + SRM + 显著性 + 分群下钻 + 归档。

【技术】

- Spring Boot 3.2
- MyBatis-Plus + MySQL
- ClickHouse（实验数据计算）
- Redis（流量分配缓存）
- Apache Commons Math（统计计算）或自实现

【核心功能】

1. 实验配置（假设 / 分流 / 主指标 / 副指标 / 护栏）
2. 流量分配（一致性哈希）
3. SRM 检查（样本比例异常）
4. 显著性计算（t-test / chi2 / Mann-Whitney U）
5. 分群下钻
6. 实验归档（→ 决策追溯库）
7. 灰度发布（按权重渐进上线）

【实验配置示例】

```yaml
experiment:
  code: EXP-001
  name: "Trial 智能引导加强"
  hypothesis: "Trial→Paid 转化率从 18% → 22%"
  related_playbook: P-SAC-001
  primary_metric: trial_to_paid_rate
  secondary_metrics: [day7_activation, sales_efficiency, arpu]
  guardrail_metrics:
    - { metric: customer_support_ticket, threshold: { op: "<=", value: 10, unit: "%" } }
    - { metric: unsubscribe_rate, threshold: { op: "<=", value: 0.5, unit: "pp" } }
  split:
    dimension: user_id
    variants:
      - { name: control, weight: 50 }
      - { name: treatment, weight: 50 }
  sample_size: 2880
  duration_days: 21
  srm_check: true
  status: running
  owner_id: 1001
  started_at: 2026-05-23T00:00:00+08:00
```

【流量分配算法】

```java
public String assignVariant(String experimentId, String entityId) {
    long hash = MurmurHash3.hash128(experimentId + ":" + entityId)[0];
    double bucket = (hash & 0xFFFF_FFFFL) * 1.0 / 0xFFFF_FFFFL;
    double accumulated = 0;
    for (Variant v : variants) {
        accumulated += v.getWeight() / 100.0;
        if (bucket < accumulated) return v.getName();
    }
    return variants.get(variants.size() - 1).getName();
}
```

确保：
- 相同 user 在同实验中始终同一组
- 不同实验间分配独立
- 缓存到 Redis 避免重复计算

【SRM 检查】

- Sample Ratio Mismatch
- 实际比例与设计比例做 chi-square test
- p < 0.001 → SRM 警告，需排查
- 每日自动检查 + 告警

【显著性计算】

支持多种检验：
- t-test（连续变量，正态分布）
- Welch's t-test（方差不等）
- Mann-Whitney U（非参数）
- chi-square test（比率类）
- Z-test（大样本比率）

输出：
- p-value
- effect size + 置信区间
- 是否显著（默认 α = 0.05）

【分群下钻】

支持按维度拆分：
- 用户规模（大/中/小）
- 行业
- 渠道
- 套餐

每个分群独立显示对比结果。

【实验归档】

实验结束自动：
1. 计算最终结果
2. 生成实验报告（PDF）
3. 登记到决策追溯库（关联 Playbook）
4. 通知 Owner

【API】

```
POST /api/experiment                    # 创建
PUT  /api/experiment/{id}/status        # start/pause/stop
GET  /api/experiment/{id}/variant       # 获取用户分组（高频接口）
GET  /api/experiment/{id}/result        # 实时结果
GET  /api/experiment/{id}/report        # 完整报告
GET  /api/experiment/{id}/srm           # SRM 检查
POST /api/experiment/{id}/finalize      # 实验结束（归档）
```

【高性能查询接口】

`GET /api/experiment/{id}/variant?entityId=xxx`
- 必须 < 50ms（业务系统高频调用）
- Redis 缓存 24h
- 缓存预热

【实验治理】

- 实验必须事前注册
- 实验结果不允许"事后挑数据"
- 同漏斗节点最多 3 个并行实验
- SRM 警告时禁止做出决策

【验证】

1. 分流一致性（用户始终同组）
2. SRM 准确性
3. 显著性计算正确性（与 R/Python 对比）
4. 高并发 variant 查询（10000 QPS）
5. 实验归档自动化

【交付物】

- 完整服务
- 实验配置后台 API
- 多种统计检验实现
- 报告生成（PDF / Markdown）
- 单元测试 ≥ 80%

【安全】

- 实验权限：仅 Owner / Admin 可修改
- 实验数据访问审计
- 用户分组不可篡改（写后只读）

现在请生成。
```
