# 指标 SQL / DSL 完整实现

> 技术栈：**ClickHouse**（行为数据 OLAP）+ **MySQL**（业务数据）。

## 数据 Schema 约定

```sql
-- 行为事件主表（ClickHouse）
CREATE TABLE dwd_event (
    event_id        UUID,
    event_code      String,
    event_time      DateTime,
    server_time     DateTime,
    user_id         String,
    anon_id         String,
    org_id          String,
    tenant_id       String,
    app_id          String,           -- 'saas' / 'ai' / 'platform'
    module          String,
    properties      Map(String, String),
    platform        String,
    sdk_version     String,
    ip              IPv4,
    geo_country     String,
    geo_city        String
) ENGINE = ReplicatedMergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (app_id, event_code, user_id, event_time);

-- 用户/企业每日维表
CREATE TABLE dim_user_daily (...);
CREATE TABLE dim_org_daily (...);
```

业务表（MySQL，CDC 同步至 ClickHouse）：
- SaaS：`t_subscription`、`t_contract`、`t_subscription_change`
- AI：`t_ai_session`、`t_ai_message`、`t_ai_session_intent`
- 平台：`t_inquiry`、`t_quote`、`t_inventory`、`t_merchant_membership`、`t_traffic_purchase`

---

## 一、SaaS LCM 指标 SQL（22 个）

### M-SA-001 · CAC
```sql
SELECT toStartOfMonth(month) AS month,
    sum(marketing_cost + sales_cost) / countDistinct(org_id) AS cac
FROM dws_finance_monthly f
LEFT JOIN (
    SELECT toStartOfMonth(min(signed_at)) AS month, org_id FROM t_contract
    WHERE arr > 0 GROUP BY org_id
) c ON f.month = c.month
GROUP BY month ORDER BY month;
```

### M-SA-002 · LTV/CAC
```sql
WITH ltv AS (
    SELECT avg(monthly_gross_margin) / avg(monthly_churn_rate) AS ltv_value
    FROM dws_saas_customer_monthly WHERE month >= addMonths(today(), -12)
)
SELECT ltv.ltv_value AS ltv, cac.cac, ltv.ltv_value / cac.cac AS ltv_cac_ratio
FROM ltv CROSS JOIN (
    SELECT avg(cac) AS cac FROM dws_cac_monthly WHERE month >= addMonths(today(), -3)
) cac;
```

### M-SA-003 · Payback Period
```sql
SELECT avg(cac) / (avg(monthly_arpu) * avg(gross_margin_rate)) AS payback_months
FROM dws_saas_unit_economics WHERE month >= addMonths(today(), -6);
```

### M-SAC-001 · Trial→Paid 转化率
```sql
WITH trial_users AS (
    SELECT user_id, min(event_time) AS trial_start FROM dwd_event
    WHERE event_code = 'saas_trial_start' AND app_id = 'saas'
        AND event_time BETWEEN '{start}' AND '{end}' GROUP BY user_id
),
paid_users AS (
    SELECT t.user_id FROM trial_users t
    INNER JOIN (
        SELECT user_id, min(event_time) AS paid_time FROM dwd_event
        WHERE event_code = 'saas_payment_success' GROUP BY user_id
    ) p ON t.user_id = p.user_id
    WHERE p.paid_time BETWEEN t.trial_start AND addDays(t.trial_start, 30)
)
SELECT count(DISTINCT p.user_id) * 1.0 / count(DISTINCT t.user_id) AS trial_to_paid_rate
FROM trial_users t LEFT JOIN paid_users p ON t.user_id = p.user_id;
```

### M-SAC-002 · TTV
```sql
SELECT
    quantileExact(0.5)(toUnixTimestamp(value_action_time) - toUnixTimestamp(signup_time)) AS ttv_median_seconds,
    quantileExact(0.75)(toUnixTimestamp(value_action_time) - toUnixTimestamp(signup_time)) AS ttv_p75
FROM (
    SELECT u.user_id, u.signup_time, min(e.event_time) AS value_action_time
    FROM dim_user_daily u INNER JOIN dwd_event e ON u.user_id = e.user_id
    WHERE e.event_code IN ('saas_contract_signed', 'saas_invoice_generated')
        AND u.signup_time BETWEEN '{start}' AND '{end}'
    GROUP BY u.user_id, u.signup_time
);
```

### M-SAC-003 · 首日激活率
```sql
SELECT countIf(activated = 1) * 1.0 / count() AS day1_activation_rate
FROM (
    SELECT u.user_id,
        if(countIf(e.event_code IN ('saas_setup_complete', 'saas_core_feature_used')
            AND e.event_time <= addDays(u.signup_time, 1)) >= 1, 1, 0) AS activated
    FROM dim_user_daily u LEFT JOIN dwd_event e ON u.user_id = e.user_id
    WHERE u.signup_time BETWEEN '{start}' AND '{end}'
    GROUP BY u.user_id, u.signup_time
);
```

### M-SAC-004 · Onboarding 完成率
```sql
SELECT countIf(completed_steps >= total_steps) * 1.0 / count() AS onboarding_completion_rate
FROM (
    SELECT user_id, countDistinct(properties['step']) AS completed_steps, 7 AS total_steps
    FROM dwd_event WHERE event_code = 'saas_onboarding_step_complete'
    GROUP BY user_id
);
```

### M-SAD-001 · 核心模块采纳率
```sql
SELECT s.module,
    countDistinct(if(e.cnt >= 5, e.org_id, NULL)) * 1.0 / countDistinct(s.org_id) AS adoption_rate
FROM t_subscription_module s
LEFT JOIN (
    SELECT org_id, module, count() AS cnt FROM dwd_event
    WHERE event_time >= addDays(today(), -30) AND app_id = 'saas'
    GROUP BY org_id, module
) e ON s.org_id = e.org_id AND s.module = e.module
WHERE s.status = 'active' AND s.module IN ('contract', 'invoice', 'crm')
GROUP BY s.module;
```

### M-SAD-002 · Seat 激活率
```sql
SELECT sum(active_seats) * 1.0 / sum(purchased_seats) AS seat_activation_rate
FROM (
    SELECT s.org_id, s.purchased_seats,
        countDistinctIf(e.user_id, e.event_time >= addDays(today(), -30)) AS active_seats
    FROM t_subscription s LEFT JOIN dwd_event e ON s.org_id = e.org_id
    WHERE s.status = 'active' GROUP BY s.org_id, s.purchased_seats
);
```

### M-SAD-003 · Adoption Score
```sql
WITH metrics AS (
    SELECT org_id,
        avgIf(adoption_rate, type='core_module') AS core_score,
        avgIf(rate, type='seat') AS seat_score,
        avgIf(usage_rate, type='advanced') AS advanced_score,
        avgIf(usage_rate, type='api') AS api_score
    FROM dws_saas_adoption_daily WHERE date = today() GROUP BY org_id
)
SELECT org_id, (core_score * 0.4 + seat_score * 0.3 + advanced_score * 0.2 + api_score * 0.1) * 100 AS adoption_score
FROM metrics;
```

### M-SAD-004 · DAU/MAU 粘性
```sql
SELECT countDistinctIf(user_id, toDate(event_time) = today()) * 1.0 /
    countDistinctIf(user_id, event_time >= addDays(today(), -30)) AS dau_mau_ratio
FROM dwd_event WHERE app_id = 'saas';
```

### M-SE-001 · NRR
```sql
WITH base AS (
    SELECT sum(mrr) AS start_mrr FROM t_subscription
    WHERE status = 'active' AND start_date <= addMonths(today(), -12)
        AND (end_date IS NULL OR end_date >= addMonths(today(), -12))
),
expansion AS (
    SELECT sum(mrr_delta) AS expansion_mrr FROM t_subscription_change
    WHERE change_type IN ('upgrade', 'add_seat', 'add_module')
        AND change_date >= addMonths(today(), -12)
),
contraction AS (
    SELECT sum(abs(mrr_delta)) AS contraction_mrr FROM t_subscription_change
    WHERE change_type = 'downgrade' AND change_date >= addMonths(today(), -12)
),
churn AS (
    SELECT sum(mrr) AS churn_mrr FROM t_subscription
    WHERE status = 'churned' AND churn_date >= addMonths(today(), -12)
)
SELECT (base.start_mrr + expansion.expansion_mrr - contraction.contraction_mrr - churn.churn_mrr) 
       * 1.0 / base.start_mrr AS nrr
FROM base CROSS JOIN expansion CROSS JOIN contraction CROSS JOIN churn;
```

### M-SE-002 · GRR
```sql
SELECT (base.start_mrr - contraction.contraction_mrr - churn.churn_mrr) * 1.0 / base.start_mrr AS grr
FROM base CROSS JOIN contraction CROSS JOIN churn;
```

### M-SE-003 · 续约率
```sql
SELECT countIf(renewal_status = 'renewed') * 1.0 / count() AS logo_renewal_rate
FROM t_contract WHERE expire_at BETWEEN '{start}' AND '{end}';
```

### M-SE-004 · 扩张率
```sql
SELECT sum(mrr_delta) / (
    SELECT sum(mrr) FROM t_subscription 
    WHERE status='active' AND start_date <= addMonths(today(), -12)
) AS expansion_rate
FROM t_subscription_change
WHERE change_type IN ('upgrade', 'add_seat', 'add_module')
    AND change_date >= addMonths(today(), -12);
```

### M-SE-005 · 续约提前指数
```sql
SELECT quantileExact(0.5)(dateDiff('day', renewed_at, expire_at)) AS renewal_lead_days_median
FROM t_contract WHERE renewal_status = 'renewed' AND renewed_at IS NOT NULL
    AND expire_at >= today();
```

### M-SR-001 · Logo 留存
```sql
SELECT countIf(paid_today = 1 AND paid_year_ago = 1) * 1.0 / countIf(paid_year_ago = 1) AS logo_retention
FROM (
    SELECT org_id,
        maxIf(1, status='active' AND date = today()) AS paid_today,
        maxIf(1, status='active' AND date = addYears(today(), -1)) AS paid_year_ago
    FROM dim_org_daily GROUP BY org_id
);
```

### M-SR-002 · 流失原因分布
```sql
SELECT churn_reason, count() AS cnt, count() * 1.0 / sum(count()) OVER () AS pct
FROM t_churn_record WHERE churn_date >= addMonths(today(), -3)
GROUP BY churn_reason ORDER BY cnt DESC;
```

### M-SR-003 · 流失预警准确率
```sql
SELECT countIf(predicted_high_risk = 1 AND actual_churned = 1) * 1.0 /
    countIf(predicted_high_risk = 1) AS precision_at_high_risk
FROM dws_churn_prediction
WHERE prediction_date BETWEEN addMonths(today(), -3) AND addMonths(today(), -1);
```

### M-SH-001 · 客户健康分
```sql
SELECT org_id,
    (adoption_score * 0.30 + activity_score * 0.20 + admin_health_score * 0.15
    + renewal_signal_score * 0.15 + csat_score * 0.10 + commercial_score * 0.10) AS health_score
FROM dws_org_health_components_daily WHERE date = today();
```

### M-SH-002 · NPS
```sql
SELECT (countIf(score >= 9) - countIf(score <= 6)) * 100.0 / count() AS nps
FROM t_nps_survey WHERE created_at BETWEEN '{start}' AND '{end}' AND app_id = 'saas';
```

### M-SH-003 · CSAT
```sql
SELECT avg(rating) AS csat FROM t_service_rating WHERE created_at BETWEEN '{start}' AND '{end}';
```

---

## 二、AI 单位经济指标 SQL（20 个）

### M-AV-001 · 价值采纳率
```sql
SELECT countIf(has_copy = 1 OR has_export = 1 OR has_thumbsup = 1 OR has_followup = 1) * 1.0 / count() AS value_adoption_rate
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AV-002 · TTFV
```sql
SELECT quantileExact(0.5)(ttfv_seconds) AS ttfv_median
FROM (
    SELECT u.user_id, min(toUnixTimestamp(s.started_at)) - toUnixTimestamp(u.signup_time) AS ttfv_seconds
    FROM dim_user_daily u INNER JOIN t_ai_session s ON u.user_id = s.user_id
    WHERE u.signup_time BETWEEN '{start}' AND '{end}' AND u.app_id = 'ai'
        AND (s.has_copy = 1 OR s.has_export = 1 OR s.has_thumbsup = 1)
    GROUP BY u.user_id, u.signup_time
);
```

### M-AV-003 · 高质量输出率
```sql
SELECT countIf(has_copy = 1 AND has_thumbsdown = 0 AND output_tokens BETWEEN 50 AND 3000) * 1.0 / count() AS high_quality_rate
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AV-004 · 单用户日均价值动作
```sql
SELECT sum(value_actions) * 1.0 / countDistinct(user_id) AS avg_value_actions_per_dau
FROM (
    SELECT user_id, toDate(started_at) AS d,
        sum(has_copy + has_export + has_thumbsup + has_followup) AS value_actions
    FROM t_ai_session WHERE toDate(started_at) = today() GROUP BY user_id, d
);
```

### M-AU-001 · 单次会话毛利
```sql
SELECT avg(revenue_per_session - cost_per_session) AS avg_session_gross_margin
FROM (
    SELECT s.session_id, s.cost AS cost_per_session,
        (monthly_subscription_revenue / monthly_session_count) AS revenue_per_session
    FROM t_ai_session s LEFT JOIN dws_user_subscription_monthly sub
        ON s.user_id = sub.user_id AND toStartOfMonth(s.started_at) = sub.month
    WHERE s.started_at BETWEEN '{start}' AND '{end}'
);
```

### M-AU-002 · 单 Token 成本
```sql
SELECT sum(cost) / (sum(input_tokens + output_tokens) / 1000.0) AS cost_per_1k_token
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AU-003 · 单次会话 Token 消耗
```sql
SELECT avg(input_tokens + output_tokens) AS avg_tokens_per_session
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AQ-001 · 点踩率
```sql
SELECT countIf(has_thumbsdown = 1) * 1.0 / count() AS thumbsdown_rate
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AQ-002 · 拒答率
```sql
SELECT countIf(properties['is_refusal'] = '1') * 1.0 / count() AS refusal_rate
FROM dwd_event WHERE event_code = 'ai_response_generated' AND event_time BETWEEN '{start}' AND '{end}';
```

### M-AQ-003 · 重写率
```sql
SELECT countDistinctIf(session_id, message_count > 1 AND properties['is_rewrite'] = '1') * 1.0 / countDistinct(session_id) AS rewrite_rate
FROM (
    SELECT session_id, count() AS message_count, any(properties) AS properties
    FROM t_ai_message WHERE role = 'user' GROUP BY session_id
);
```

### M-AQ-004 · 能力缺口热度
```sql
SELECT intent_cluster, count() AS sessions,
    avgIf(1, has_thumbsdown=1) AS thumbsdown_rate,
    avgIf(1, has_copy=1 OR has_export=1) AS adoption_rate
FROM t_ai_session s JOIN t_ai_session_intent i ON s.session_id = i.session_id
WHERE s.started_at >= addDays(today(), -30)
GROUP BY intent_cluster
HAVING sessions >= 100
ORDER BY thumbsdown_rate DESC;
```

### M-AR-001 · 7 日复用
```sql
SELECT countDistinctIf(user_id, day7_active = 1) * 1.0 / countDistinct(user_id) AS d7_reuse_rate
FROM (
    SELECT s.user_id, min(toDate(s.started_at)) AS first_day,
        maxIf(1, toDate(s2.started_at) BETWEEN addDays(min(toDate(s.started_at)), 1) AND addDays(min(toDate(s.started_at)), 7)) AS day7_active
    FROM t_ai_session s LEFT JOIN t_ai_session s2 ON s.user_id = s2.user_id
    WHERE s.started_at BETWEEN '{start}' AND '{end}' GROUP BY s.user_id
);
```

### M-AR-002 · D30 留存
```sql
SELECT countDistinctIf(user_id, d30_active = 1) * 1.0 / countDistinct(user_id) AS d30_retention
FROM (
    SELECT first.user_id,
        maxIf(1, toDate(s.started_at) = addDays(first.first_day, 30)) AS d30_active
    FROM (SELECT user_id, min(toDate(started_at)) AS first_day FROM t_ai_session GROUP BY user_id) first
    LEFT JOIN t_ai_session s ON first.user_id = s.user_id
    WHERE first.first_day BETWEEN addDays(today(), -60) AND addDays(today(), -30)
    GROUP BY first.user_id, first.first_day
);
```

### M-AR-003 · 月会话数
```sql
SELECT count() * 1.0 / countDistinct(user_id) AS sessions_per_mau
FROM t_ai_session WHERE started_at >= addDays(today(), -30);
```

### M-AC-001 · 免费→付费转化率
```sql
SELECT countDistinctIf(user_id, was_paid=1) * 1.0 / countDistinct(user_id) AS free_to_paid_rate
FROM (
    SELECT u.user_id,
        maxIf(1, sub.plan != 'free' AND sub.start_date BETWEEN u.signup_time AND addDays(u.signup_time, 90)) AS was_paid
    FROM dim_user_daily u LEFT JOIN t_subscription sub ON u.user_id = sub.user_id
    WHERE u.app_id='ai' AND u.signup_time BETWEEN addDays(today(), -120) AND addDays(today(), -90)
    GROUP BY u.user_id, u.signup_time
);
```

### M-AC-002 · ARPU
```sql
SELECT sum(mrr) / countDistinct(user_id) AS arpu
FROM t_subscription WHERE app_id='ai' AND plan != 'free' AND status='active';
```

### M-AC-003 · 升级率
```sql
SELECT countIf(change_type='upgrade') * 1.0 / countDistinct(user_id) AS upgrade_rate
FROM t_subscription_change WHERE app_id='ai' AND change_date BETWEEN '{start}' AND '{end}';
```

### M-AE-001 · 路由命中率
```sql
SELECT countIf(routed_to_small_model = 1) * 1.0 / count() AS routing_hit_rate
FROM t_ai_session
WHERE properties['is_simple_request'] = '1' AND started_at BETWEEN '{start}' AND '{end}';
```

### M-AE-002 · 缓存命中率
```sql
SELECT countIf(cache_hit = 1) * 1.0 / count() AS cache_hit_rate
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

### M-AE-003 · 延迟 P95
```sql
SELECT quantileExact(0.95)(latency_ms) AS latency_p95
FROM t_ai_session WHERE started_at BETWEEN '{start}' AND '{end}';
```

---

## 三、平台双边流动性指标 SQL（18 个）

### M-L-001 · 24h 响应率
```sql
SELECT countIf(first_response_at IS NOT NULL 
    AND first_response_at - created_at <= INTERVAL 24 HOUR) * 1.0 / count() AS response_rate_24h
FROM t_inquiry WHERE created_at BETWEEN '{start}' AND '{end}';
```

### M-L-002 · 撮合成功率
```sql
SELECT countIf(status='success') * 1.0 / count() AS match_success_rate
FROM t_inquiry 
WHERE created_at BETWEEN addDays('{start}', -30) AND addDays('{end}', -30);
```

### M-L-003 · 平均撮合时长
```sql
SELECT quantileExact(0.5)(toUnixTimestamp(deal_at) - toUnixTimestamp(created_at)) / 86400.0 AS match_days_median
FROM t_inquiry WHERE status='success' AND created_at BETWEEN '{start}' AND '{end}';
```

### M-L-004 · 询盘活跃度
```sql
SELECT count() / dateDiff('day', '{start}', '{end}') AS daily_inquiries
FROM t_inquiry WHERE created_at BETWEEN '{start}' AND '{end}';
```

### M-L-005 · 响应时长分布
```sql
SELECT
    countIf(response_hours <= 1) * 1.0 / count() AS within_1h,
    countIf(response_hours <= 6) * 1.0 / count() AS within_6h,
    countIf(response_hours <= 24) * 1.0 / count() AS within_24h,
    countIf(response_hours > 24) * 1.0 / count() AS over_24h
FROM (
    SELECT inquiry_id, (toUnixTimestamp(first_response_at) - toUnixTimestamp(created_at)) / 3600.0 AS response_hours
    FROM t_inquiry WHERE first_response_at IS NOT NULL AND created_at BETWEEN '{start}' AND '{end}'
);
```

### M-B-001 · 类目供需比
```sql
SELECT i.category,
    count(DISTINCT inv.sku_id) AS supply_count,
    count(DISTINCT i.inquiry_id) AS demand_count,
    count(DISTINCT inv.sku_id) * 1.0 / count(DISTINCT i.inquiry_id) AS supply_demand_ratio
FROM t_inventory inv FULL OUTER JOIN t_inquiry i ON inv.category = i.category
WHERE (inv.status='active') AND (i.created_at >= addDays(today(), -30))
GROUP BY i.category;
```

### M-B-002 · 双边活跃比
```sql
SELECT
    countDistinctIf(user_id, app_id='platform' AND properties['role']='buyer') * 1.0 /
    countDistinctIf(user_id, app_id='platform' AND properties['role']='merchant') AS buyer_merchant_ratio
FROM dwd_event WHERE toDate(event_time) = today();
```

### M-B-003 · 供需热度地图
```sql
SELECT i.category, i.region, count(inv.sku_id) AS supply, count(i.inquiry_id) AS demand
FROM t_inventory inv FULL OUTER JOIN t_inquiry i 
    ON inv.category = i.category AND inv.region = i.region
WHERE inv.status='active' AND i.created_at >= addDays(today(), -30)
GROUP BY i.category, i.region;
```

### M-R-001 · 买家次月留存
```sql
SELECT countDistinct(b1.buyer_id) * 1.0 / (
    SELECT countDistinct(buyer_id) FROM t_inquiry 
    WHERE toStartOfMonth(created_at)=addMonths(toStartOfMonth(now()),-1)
) AS buyer_retention
FROM t_inquiry b1
WHERE toStartOfMonth(b1.created_at) = toStartOfMonth(now())
    AND b1.buyer_id IN (
        SELECT buyer_id FROM t_inquiry 
        WHERE toStartOfMonth(created_at)=addMonths(toStartOfMonth(now()),-1)
    );
```

### M-R-002 · 商家续费率
```sql
SELECT countIf(renewal_status='renewed') * 1.0 / count() AS merchant_renewal_rate
FROM t_merchant_membership WHERE expire_date BETWEEN '{start}' AND '{end}';
```

### M-R-003 · 买家复购周期
```sql
SELECT quantileExact(0.5)(date_diff_days) AS repurchase_interval_median
FROM (
    SELECT buyer_id, toDate(created_at) AS d,
        dateDiff('day', lagInFrame(toDate(created_at), 1) OVER (PARTITION BY buyer_id ORDER BY created_at), toDate(created_at)) AS date_diff_days
    FROM t_inquiry
) WHERE date_diff_days > 0;
```

### M-N-001 · 商家弹性系数
```sql
WITH monthly AS (
    SELECT toStartOfMonth(d) AS m,
        countDistinctIf(merchant_id, new_merchant=1) AS new_merchants_100s,
        countDistinct(buyer_id) AS active_buyers
    FROM dws_platform_daily GROUP BY m
)
SELECT m, new_merchants_100s, active_buyers,
    active_buyers - lagInFrame(active_buyers) OVER (ORDER BY m) AS buyer_delta
FROM monthly ORDER BY m;
```

### M-M-001 · GMV
```sql
SELECT sum(gmv) AS gmv FROM t_inquiry WHERE status='success' AND deal_at BETWEEN '{start}' AND '{end}';
```

### M-M-002 · Take Rate
```sql
SELECT sum(platform_revenue) / sum(gmv) AS take_rate
FROM t_inquiry WHERE status='success' AND deal_at BETWEEN '{start}' AND '{end}';
```

### M-M-003 · 会员 ARPU
```sql
SELECT sum(membership_fee) / countDistinct(merchant_id) AS membership_arpu
FROM t_merchant_membership WHERE start_date <= '{end}' AND end_date >= '{start}';
```

### M-M-004 · 流量包 ARPU
```sql
SELECT sum(amount) / countDistinct(merchant_id) AS traffic_pkg_arpu
FROM t_traffic_purchase WHERE purchase_at BETWEEN '{start}' AND '{end}';
```

### M-M-005 · 流量包 ROI
```sql
SELECT merchant_id, sum(amount) AS pkg_cost, sum(gmv_from_pkg) AS pkg_gmv,
    sum(gmv_from_pkg) * gross_margin / sum(amount) AS roi
FROM t_traffic_attribution
WHERE purchase_at BETWEEN '{start}' AND '{end}' GROUP BY merchant_id;
```

---

## 治理规范

| 项 | 规范 |
|---|---|
| 注册 | SQL 必须入指标平台统一管理 |
| 校验 | 上线前对比 3 个历史时段 |
| 性能 | P95 ≤ 5s，复杂指标走物化视图 |
| 版本 | 变更需评审 + 版本号 |
| 文档 | 每条 SQL 配业务定义 + 字段说明 + 责任人 |
