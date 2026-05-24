# 埋点规范与标准事件清单

## 一、命名规范

### 事件命名

```
{产品code}_{模块}_{对象}_{动作}

全小写 + 下划线
最长 60 字符
必须在元数据中心注册才可上报

示例：
  saas_invoice_export_click
  saas_contract_signed
  ai_chat_session_start
  ai_response_thumbsup
  platform_inquiry_create
  platform_quote_submit
  platform_search_zero_result
```

### 字段命名

```
snake_case
预留系统字段（不可被业务覆盖）：
  event_id          雪花 ID
  event_code        事件 code
  event_time        客户端时间
  server_time       服务端时间
  user_id           用户 ID
  anon_id           匿名 ID
  org_id            企业 ID
  tenant_id         租户
  app_id            产品标识 'saas' / 'ai' / 'platform'
  sdk_version       SDK 版本
  platform          'web' / 'ios' / 'android' / 'wxapp' / 'server'
  ip                IP（脱敏）
  geo_country / geo_city  地理位置
  ua_browser / ua_os
```

### 业务属性（properties）

- 全小写 + 下划线
- 类型：string / int / double / boolean / array
- 单个属性 ≤ 1KB
- 单事件 properties ≤ 64KB

## 二、事件元数据注册流程

```
PM 提需求 → 数据 PM 审核 → 元数据中心注册 → 发布到 SDK 文档 → 业务方接入
   ↓
注册内容：
  · event_code（必填）
  · event_name（中文）
  · app_id / module
  · description
  · 必填字段定义（含类型 + 约束）
  · 可选字段定义
  · 责任 PM
  · 关联指标（如有）
  · 数据保留策略

未注册事件被 Collector 拒绝（或进异常队列）。
```

## 三、SDK 接入 SOP

### 业务方接入 5 步

```
1. 与数据 PM 确认本产品的事件清单
   ↓
2. 元数据中心注册事件
   ↓
3. 集成 SDK（Java / Vue3）
   ↓
4. 在代码中上报事件
   ↓
5. 沙箱环境验证 → 切到生产
```

### 验收 Checklist

- [ ] 所有核心业务事件已埋点
- [ ] 上报成功率 ≥ 99%
- [ ] 数据准确性（与业务库对比）
- [ ] 字段完整率 ≥ 95%
- [ ] 无未注册事件
- [ ] 用户隐私字段已脱敏

## 四、标准事件清单（按产品）

### 通用事件（所有产品）

| event_code | 说明 | 必填字段 |
|---|---|---|
| `user_signup` | 注册 | source, channel |
| `user_login` | 登录 | method |
| `user_logout` | 登出 | — |
| `page_view` | 页面浏览 | url, title, referrer |
| `page_leave` | 页面离开 | url, duration_ms |
| `feature_click` | 功能点击 | feature_code |
| `error_occurred` | 错误发生 | error_type, error_msg, stack |
| `nps_submit` | NPS 提交 | score, comment |
| `feedback_submit` | 反馈提交 | type, content |

### SaaS 专属

| event_code | 说明 | 必填字段 |
|---|---|---|
| `saas_trial_start` | 试用启动 | plan, source |
| `saas_payment_success` | 付费成功 | plan, amount, payment_method |
| `saas_payment_failed` | 付费失败 | plan, reason |
| `saas_subscription_renew` | 续约 | plan, amount |
| `saas_subscription_cancel` | 取消订阅 | reason |
| `saas_subscription_upgrade` | 升级套餐 | from_plan, to_plan |
| `saas_subscription_downgrade` | 降级套餐 | from_plan, to_plan |
| `saas_seat_invite` | 邀请席位 | invitee_email, role |
| `saas_seat_activate` | 席位激活 | invited_by |
| `saas_setup_complete` | 完成基础设置 | — |
| `saas_onboarding_step_complete` | 完成新手引导步骤 | step |
| `saas_core_feature_used` | 使用核心功能 | feature_code |
| `saas_module_enable` | 启用模块 | module_code |
| `saas_module_first_use` | 模块首次使用 | module_code |
| `saas_invoice_create` | 创建发票 | amount, customer |
| `saas_invoice_export` | 导出发票 | format, count |
| `saas_contract_create` | 创建合同 | type, amount |
| `saas_contract_signed` | 合同签字 | contract_id |
| `saas_data_export` | 数据导出 | type, count, format |
| `saas_api_call` | API 调用（高级功能） | endpoint |

### AI 专属

| event_code | 说明 | 必填字段 |
|---|---|---|
| `ai_session_start` | 会话开始 | session_id, scene |
| `ai_session_end` | 会话结束 | session_id, duration_ms |
| `ai_prompt_submit` | Prompt 提交 | session_id, prompt_length, intent? |
| `ai_response_generated` | 输出生成 | session_id, model, input_tokens, output_tokens, cost, latency_ms |
| `ai_response_copy` | 输出复制 | session_id |
| `ai_response_export` | 输出导出 | session_id, format |
| `ai_response_thumbsup` | 点赞 | session_id |
| `ai_response_thumbsdown` | 点踩 | session_id, reason? |
| `ai_response_rewrite` | 用户改写 prompt 重来 | session_id |
| `ai_followup_question` | 继续追问 | session_id |
| `ai_template_use` | 使用模板 | template_id |
| `ai_model_switch` | 切换模型 | from_model, to_model |
| `ai_quota_warning` | 配额警告 | usage, quota |
| `ai_quota_exceed` | 配额超出 | quota |

### 平台买家侧

| event_code | 说明 | 必填字段 |
|---|---|---|
| `platform_search` | 搜索 | query, category, filters |
| `platform_search_zero_result` | 搜索零结果 | query, category |
| `platform_product_view` | 商品详情查看 | sku_id, merchant_id |
| `platform_product_favorite` | 收藏 | sku_id |
| `platform_inquiry_create` | 发起询盘 | inquiry_id, category, sku_id?, merchants_count |
| `platform_inquiry_received_quote` | 收到报价 | inquiry_id, quote_id, merchant_id, price |
| `platform_inquiry_select_quote` | 选择报价（成交） | inquiry_id, quote_id, gmv |
| `platform_lead_submit` | 留线索 | category, contact |
| `platform_chat_send` | 发起对话 | merchant_id |
| `platform_review_submit` | 提交评价 | merchant_id, rating |

### 平台商家侧

| event_code | 说明 | 必填字段 |
|---|---|---|
| `platform_merchant_register` | 商家注册 | category |
| `platform_inventory_publish` | 库存发布 | sku_id, category |
| `platform_inventory_update` | 库存更新 | sku_id |
| `platform_inquiry_received` | 收到询盘 | inquiry_id, buyer_id |
| `platform_quote_submit` | 提交报价 | quote_id, inquiry_id, price |
| `platform_chat_response` | 响应对话 | buyer_id, response_time_seconds |
| `platform_membership_purchase` | 购买会员 | level, amount |
| `platform_traffic_pkg_purchase` | 购买流量包 | pkg_id, amount |
| `platform_buyer_view_contact` | 查看买家联系方式 | buyer_id |

## 五、数据质量规则

### 必检

- [ ] 必填字段完整率 ≥ 99%
- [ ] 时间字段无异常（无负值、无未来时间）
- [ ] user_id 与 OneID 可关联
- [ ] event_time 与 server_time 漂移 ≤ 7 天
- [ ] 字段类型与定义一致
- [ ] 枚举字段值在合法范围
- [ ] 数值字段在合理区间

### 异常处理

```
正常事件 → events_raw → dwd_event
   ↓
缺失必填 / 字段类型错误 / 未注册事件 → DLQ + 通知数据 PM
   ↓
时间漂移过大 → 隔离区 + 日报
   ↓
重复事件（event_id 重复）→ 去重保留首条
```

## 六、隐私与合规

### 不可上报的字段

- 完整身份证号
- 完整银行卡号
- 密码 / Token
- 用户裸文本内容（如对话内容）—— 改为脱敏后摘要

### 必须脱敏

- 手机号：保留前 3 后 4
- 邮箱：保留首字母 + 域名
- 姓名：保留姓
- IP：保留 IP 段

### 用户授权

- 注册时明确告知埋点 + 用户同意
- 提供"关闭埋点"开关（PIPL 要求）
- 关闭后不上报行为数据，仅必要业务事件

## 七、与指标的对应

```
事件 → 指标计算的原料

例：
  saas_payment_success → MRR / ARR / 新增付费数
  saas_invoice_export → 发票模块采纳
  ai_response_copy → 价值采纳率
  platform_inquiry_create → 询盘活跃度
  platform_inquiry_received_quote → 24h 响应率（结合时间）
```

每个核心指标必须有对应埋点。指标定义时关联事件 code。

## 八、版本管理

### 事件 Schema 演进

- 新增字段：向后兼容，无需新版本
- 改字段类型：需新版本，老字段保留兼容期 ≥ 6 月
- 删除字段：标记 deprecated，6 月后下线

### 事件下线流程

```
1. 数据 PM 提议下线（评估使用情况）
2. 关联 PM 评审
3. 通知业务方
4. 在元数据中心标记 deprecated
5. 6 个月后从生产环境拒绝
```

## 九、最佳实践

### Do
- ✅ 事件命名清晰、动宾结构
- ✅ 关键业务都埋点（即使现在没用，将来可能要）
- ✅ 维度字段尽量打全（事后想拆解就有数据）
- ✅ 使用枚举字段而非自由文本（便于聚合）
- ✅ 同一行为只埋一次（避免重复计数）

### Don't
- ❌ 事件命名模糊（如 `click1`, `action`）
- ❌ properties 装大段 JSON（拆开成单独字段）
- ❌ 同一行为多个事件名（不一致）
- ❌ 漏埋核心业务事件
- ❌ 上报敏感信息

## 十、SDK 文档（业务方面向）

- 事件清单完整文档（自动生成）
- 接入示例
- 调试工具（事件查看器）
- 常见问题

## 十一、治理

- 每月数据质量报告
- 季度埋点评审（淘汰无用埋点 + 补缺核心埋点）
- 年度埋点规范升级
- 业务方接入工时统计（目标 ≤ 3 工作日）
