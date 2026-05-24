# 数据 Schema 设计

## 一、MySQL 元数据库设计

### 通用字段规约（所有表必备）

```sql
id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT  -- 雪花算法生成
tenant_id       BIGINT UNSIGNED NOT NULL                    -- 租户隔离
gmt_create      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
gmt_modified    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
creator_id      BIGINT
modifier_id     BIGINT
is_deleted      TINYINT NOT NULL DEFAULT 0
version         INT NOT NULL DEFAULT 0                      -- 乐观锁
```

**索引规范**：
- 主键 `id`
- `tenant_id` 必加索引或联合索引
- 业务唯一键加 unique
- 高频查询字段加普通索引

### 1. 元数据库（analytics_meta）

```sql
-- 事件元数据
CREATE TABLE t_meta_event (
    id BIGINT UNSIGNED PRIMARY KEY,
    event_code VARCHAR(100) NOT NULL UNIQUE,    -- 如 saas_invoice_export_click
    event_name VARCHAR(200) NOT NULL,
    app_id VARCHAR(50) NOT NULL,
    module VARCHAR(100),
    description TEXT,
    owner_id BIGINT,
    status TINYINT,                              -- 0草稿 1启用 2弃用
    schema_def JSON,                             -- 字段定义
    tenant_id BIGINT UNSIGNED NOT NULL,
    gmt_create DATETIME NOT NULL,
    gmt_modified DATETIME NOT NULL,
    creator_id BIGINT, modifier_id BIGINT,
    is_deleted TINYINT DEFAULT 0,
    version INT DEFAULT 0,
    INDEX idx_tenant_app (tenant_id, app_id, status)
);

-- 指标元数据
CREATE TABLE t_meta_metric (
    id BIGINT UNSIGNED PRIMARY KEY,
    metric_code VARCHAR(100) NOT NULL UNIQUE,   -- M-SA-001
    metric_name VARCHAR(200) NOT NULL,
    category VARCHAR(50),
    business_def TEXT NOT NULL,
    calc_formula TEXT NOT NULL,
    sql_template LONGTEXT,
    dimensions JSON,
    threshold JSON,                              -- {green:..., yellow:..., red:...}
    update_freq VARCHAR(20),
    owner_id BIGINT,
    status TINYINT,
    related_playbooks JSON,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT,
    INDEX idx_category (category, status)
);

-- 标签元数据
CREATE TABLE t_meta_tag (
    id BIGINT UNSIGNED PRIMARY KEY,
    tag_code VARCHAR(100) NOT NULL UNIQUE,
    tag_name VARCHAR(200) NOT NULL,
    tag_type ENUM('base','behavior','business','predict'),
    entity_type ENUM('user','org','merchant','buyer'),
    value_type ENUM('boolean','string','int','double','enum'),
    enum_values JSON,
    production_method ENUM('rule','sql','model'),
    production_config LONGTEXT,
    update_freq VARCHAR(20),
    owner_id BIGINT,
    status TINYINT,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT
);
```

### 2. 身份库（analytics_auth）

```sql
-- OneID 映射
CREATE TABLE t_oneid_mapping (
    id BIGINT UNSIGNED PRIMARY KEY,
    one_id VARCHAR(64) NOT NULL,                -- 统一 ID
    id_type ENUM('device','account','org','merchant','buyer','ai_session'),
    id_value VARCHAR(128) NOT NULL,
    app_id VARCHAR(50),
    confidence DECIMAL(3,2),                    -- 映射置信度 0-1
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    is_deleted TINYINT,
    UNIQUE KEY uk_id (id_type, id_value, app_id),
    INDEX idx_one (one_id)
);

-- 用户主表
CREATE TABLE t_user (
    id BIGINT UNSIGNED PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL UNIQUE,
    name VARCHAR(100), email VARCHAR(200), phone VARCHAR(20),
    org_id VARCHAR(64), role VARCHAR(50),
    status TINYINT, last_login_at DATETIME,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT,
    INDEX idx_org (org_id), INDEX idx_email (email)
);

-- 角色与权限
CREATE TABLE t_role (...);
CREATE TABLE t_permission (...);
CREATE TABLE t_user_role (...);
CREATE TABLE t_role_permission (...);

-- 审计日志
CREATE TABLE t_audit_log (
    id BIGINT UNSIGNED PRIMARY KEY,
    user_id VARCHAR(64), action VARCHAR(100),
    resource_type VARCHAR(50), resource_id VARCHAR(100),
    ip VARCHAR(50), user_agent VARCHAR(500),
    request_body LONGTEXT, response_status INT,
    trace_id VARCHAR(64),
    tenant_id BIGINT, gmt_create DATETIME,
    INDEX idx_user_time (user_id, gmt_create),
    INDEX idx_resource (resource_type, resource_id, gmt_create)
) PARTITION BY RANGE (TO_DAYS(gmt_create)) (
    -- 按月分区，自动归档
);
```

### 3. 分群库（analytics_segment）

```sql
CREATE TABLE t_seg_segment (
    id BIGINT UNSIGNED PRIMARY KEY,
    segment_code VARCHAR(100) UNIQUE,
    segment_name VARCHAR(200),
    description TEXT,
    entity_type ENUM('user','org','merchant','buyer'),
    rule_dsl LONGTEXT,                          -- 规则 DSL
    estimated_count BIGINT,
    last_snapshot_at DATETIME,
    status TINYINT,
    refresh_strategy ENUM('manual','daily','realtime'),
    owner_id BIGINT,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT
);

CREATE TABLE t_seg_snapshot (
    id BIGINT UNSIGNED PRIMARY KEY,
    segment_id BIGINT,
    snapshot_date DATE,
    user_count BIGINT,
    snapshot_url VARCHAR(500),                  -- OSS 路径
    tenant_id BIGINT, gmt_create DATETIME,
    UNIQUE KEY uk_seg_date (segment_id, snapshot_date)
);
```

### 4. Journey 与触达库（analytics_reach）

```sql
CREATE TABLE t_journey (
    id BIGINT UNSIGNED PRIMARY KEY,
    journey_code VARCHAR(100) UNIQUE,
    journey_name VARCHAR(200),
    description TEXT,
    trigger_type ENUM('event','tag_change','scheduled','manual'),
    trigger_config JSON,
    nodes_config LONGTEXT,                      -- 节点流程图 JSON
    status ENUM('draft','active','paused','archived'),
    owner_id BIGINT,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT
);

CREATE TABLE t_journey_run (
    id BIGINT UNSIGNED PRIMARY KEY,
    journey_id BIGINT, user_id VARCHAR(64),
    current_node VARCHAR(100), status ENUM('running','completed','dropped','failed'),
    entered_at DATETIME, completed_at DATETIME,
    context_data JSON,
    tenant_id BIGINT, gmt_create DATETIME,
    INDEX idx_journey_status (journey_id, status),
    INDEX idx_user (user_id, gmt_create)
);

CREATE TABLE t_reach_task (
    id BIGINT UNSIGNED PRIMARY KEY,
    journey_run_id BIGINT,
    channel ENUM('inapp','email','sms','push','wecom','dingtalk'),
    template_id BIGINT,
    target_user_id VARCHAR(64),
    payload JSON,
    status ENUM('pending','sent','delivered','opened','clicked','failed'),
    scheduled_at DATETIME, sent_at DATETIME, delivered_at DATETIME,
    error_msg TEXT,
    tenant_id BIGINT, gmt_create DATETIME,
    INDEX idx_user_time (target_user_id, scheduled_at)
);

CREATE TABLE t_reach_unsubscribe (
    id BIGINT UNSIGNED PRIMARY KEY,
    user_id VARCHAR(64),
    channel VARCHAR(50),
    category VARCHAR(50),                       -- 营销/系统/产品
    unsubscribed_at DATETIME,
    tenant_id BIGINT,
    UNIQUE KEY uk_user_channel_cat (user_id, channel, category)
);
```

### 5. 实验库（analytics_experiment）

```sql
CREATE TABLE t_experiment (
    id BIGINT UNSIGNED PRIMARY KEY,
    experiment_code VARCHAR(100) UNIQUE,
    experiment_name VARCHAR(200),
    hypothesis TEXT,
    primary_metric VARCHAR(100),
    secondary_metrics JSON,
    guardrail_metrics JSON,
    split_dimension ENUM('user','session','org','request'),
    split_config JSON,                          -- {control: 50, treatment: 50}
    sample_size BIGINT,
    started_at DATETIME, ended_at DATETIME,
    status ENUM('draft','running','paused','completed','cancelled'),
    owner_id BIGINT,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT
);

CREATE TABLE t_experiment_assignment (
    id BIGINT UNSIGNED PRIMARY KEY,
    experiment_id BIGINT, entity_id VARCHAR(64),
    variant VARCHAR(50),
    assigned_at DATETIME,
    tenant_id BIGINT,
    UNIQUE KEY uk_exp_entity (experiment_id, entity_id)
);

CREATE TABLE t_experiment_result (
    experiment_id BIGINT, variant VARCHAR(50), metric_code VARCHAR(100),
    metric_value DOUBLE, sample_size BIGINT,
    p_value DOUBLE, confidence_interval JSON,
    is_significant BOOLEAN,
    calculated_at DATETIME,
    PRIMARY KEY (experiment_id, variant, metric_code)
);
```

### 6. 决策追溯库（analytics_decision）

```sql
CREATE TABLE t_decision (
    id BIGINT UNSIGNED PRIMARY KEY,
    decision_code VARCHAR(100) UNIQUE,
    title VARCHAR(200),
    type ENUM('product','resource','strategy','organization','emergency'),
    importance ENUM('P0','P1','P2'),
    decided_by BIGINT,
    decided_at DATETIME,
    related_model ENUM('saas','ai','platform','cross'),
    related_metrics JSON,
    related_playbook VARCHAR(100),
    related_hypothesis VARCHAR(100),
    related_orgs JSON,
    related_pms JSON,
    background TEXT,
    options_json JSON,
    chosen_option VARCHAR(50),
    rationale TEXT,
    expected_impact JSON,                       -- {30d:{kpi,value},60d:...}
    review_dates JSON,
    status ENUM('pending','in_progress','succeeded','partial','failed','cancelled'),
    snapshot_url VARCHAR(500),
    visibility ENUM('private','team','department','company'),
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT,
    INDEX idx_decider_time (decided_by, decided_at),
    INDEX idx_type_status (type, status, decided_at),
    INDEX idx_model (related_model, decided_at)
);

CREATE TABLE t_decision_review (
    id BIGINT UNSIGNED PRIMARY KEY,
    decision_id BIGINT,
    review_phase ENUM('30d','60d','90d','180d','adhoc'),
    review_date DATE,
    actual_impact JSON,
    deviation_analysis TEXT,
    conclusion ENUM('succeeded','partial','failed','in_progress'),
    lessons_learned TEXT,
    knowledge_tags JSON,
    shareable BOOLEAN,
    reviewed_by BIGINT,
    reviewed_at DATETIME,
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    UNIQUE KEY uk_decision_phase (decision_id, review_phase)
);

CREATE TABLE t_decision_snapshot (
    id BIGINT UNSIGNED PRIMARY KEY,
    decision_id BIGINT,
    snapshot_type ENUM('dashboard','metric','custom'),
    snapshot_name VARCHAR(200),
    snapshot_data LONGTEXT,
    image_url VARCHAR(500),
    sql_queries TEXT,
    snapshot_at DATETIME,
    tenant_id BIGINT, gmt_create DATETIME
);

CREATE TABLE t_strategic_hypothesis (
    id BIGINT UNSIGNED PRIMARY KEY,
    hypothesis_code VARCHAR(100) UNIQUE,
    title VARCHAR(200),
    description TEXT,
    proposer_id BIGINT, owner_id BIGINT,
    target_year YEAR,
    deadline DATE,
    north_star_metric VARCHAR(100),
    target_value DOUBLE,
    current_value DOUBLE,
    leading_indicators JSON,
    milestones JSON,
    status ENUM('proposed','active','succeeded','partial','failed','cancelled'),
    tenant_id BIGINT, gmt_create DATETIME, gmt_modified DATETIME,
    creator_id BIGINT, modifier_id BIGINT, is_deleted TINYINT, version INT
);
```

### 7. AI 助理库（analytics_assistant）

```sql
CREATE TABLE t_assistant_conversation (
    id BIGINT UNSIGNED PRIMARY KEY,
    conversation_id VARCHAR(64) UNIQUE,
    user_id VARCHAR(64),
    context_type VARCHAR(50),
    context_id VARCHAR(100),
    started_at DATETIME, ended_at DATETIME,
    tenant_id BIGINT, gmt_create DATETIME
);

CREATE TABLE t_assistant_message (
    id BIGINT UNSIGNED PRIMARY KEY,
    conversation_id VARCHAR(64),
    role ENUM('user','assistant','system','tool'),
    content LONGTEXT,
    tools_called JSON,
    tokens_input INT, tokens_output INT,
    latency_ms INT,
    model VARCHAR(50),
    user_feedback ENUM('up','down','neutral'),
    feedback_text TEXT,
    created_at DATETIME,
    INDEX idx_conv (conversation_id, created_at)
);
```

## 二、ClickHouse 数据仓库

### dwd 明细层

```sql
CREATE TABLE dwd_event (
    event_id UUID,
    event_code String,
    event_time DateTime,
    server_time DateTime,
    user_id String,
    anon_id String,
    org_id String,
    tenant_id String,
    app_id String,
    module String,
    properties Map(String, String),
    platform String,
    sdk_version String,
    ip IPv4,
    geo_country String,
    geo_city String,
    ua_browser String,
    ua_os String
) ENGINE = ReplicatedMergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (app_id, event_code, user_id, event_time)
TTL event_time + INTERVAL 24 MONTH;
```

### dws 聚合层

```sql
-- 用户每日聚合
CREATE TABLE dws_user_event_daily (
    date Date,
    user_id String, org_id String, app_id String,
    event_code String,
    event_count UInt64,
    unique_session UInt64,
    first_event_time DateTime, last_event_time DateTime,
    tenant_id String
) ENGINE = ReplicatedSummingMergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (app_id, date, user_id, event_code)
TTL date + INTERVAL 36 MONTH;

-- 企业每日聚合
CREATE TABLE dws_org_daily (...);

-- 物化视图（实时聚合 DAU/PV）
CREATE MATERIALIZED VIEW mv_dau_daily TO dws_dau_daily AS
SELECT toDate(event_time) AS date, app_id, uniqState(user_id) AS dau_state
FROM dwd_event GROUP BY date, app_id;
```

### ads 应用层

```sql
-- 客户健康分（每日全量）
CREATE TABLE ads_org_health (
    date Date, org_id String,
    health_score Decimal(5,2),
    adoption_score Decimal(5,2),
    activity_score Decimal(5,2),
    admin_health_score Decimal(5,2),
    renewal_signal_score Decimal(5,2),
    csat_score Decimal(5,2),
    commercial_score Decimal(5,2),
    risk_level Enum('green'=1,'yellow'=2,'red'=3),
    tenant_id String
) ENGINE = ReplicatedReplacingMergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (date, org_id);
```

## 三、Redis Key 规范

```
{biz}:{module}:{id}              # 业务键
{biz}:cache:{type}:{key}         # 缓存
{biz}:lock:{resource}            # 分布式锁
{biz}:queue:{name}               # 队列

例：
analytics:metric:cache:M-SA-001:hash(params)   TTL 5min
analytics:tag:user:{user_id}                    TTL 1h
analytics:auth:token:{token}                    TTL 1h
analytics:fatigue:user:{user_id}:24h            TTL 24h
analytics:idempotency:{key}                     TTL 24h
```

规则：
- 所有 key 必须设 TTL
- 单 key value ≤ 5MB
- Hash / Set / ZSet 元素 ≤ 5000

## 四、Kafka Topic 规范

```
events_raw                       # 原始事件（按 app_id 分区，32 partitions）
events_validated                 # 校验后事件
biz_cdc                          # 业务库 CDC
journey_trigger                  # Journey 触发
reach_task                       # 触达任务
insight_anomaly                  # 异动事件
dlq_events                       # 死信队列
```

- 副本数 ≥ 3
- 分区数按吞吐预估
- 消息保留 ≥ 7 天

## 五、对象存储（OSS/MinIO）

```
bucket: analytics-platform-prod
├── exports/{tenant_id}/{date}/{report_id}.xlsx
├── snapshots/{decision_id}/{type}.png
├── snapshots/{decision_id}/data.json
├── session-replay/{session_id}.json
└── sdk-releases/{version}/
```

- 跨区备份
- 生命周期：3 个月后转低频，1 年后转归档
- 决策快照永久保留
