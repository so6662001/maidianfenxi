# 提示词 11 · 决策助理（LLM 应用）

> 前置：`00-master-prompt.md`。参考：`docs/07-ai-assistant/`。

---

## 提示词正文

```
你是 AI 应用工程师。请开发 analytics-assistant 服务（LLM 决策助理）。

【目标】
对话式决策助理 L1-L4 能力，统一中台入口，每次回答含"数据+解释+行动"。

【技术】

- Spring Boot 3.2
- LangChain4j（LLM 框架）
- 主 LLM：OpenAI GPT-4 / Claude / 通义千问 / 文心一言（多 Provider）
- 小模型：GPT-4o-mini / Claude Haiku / Qwen-Turbo
- Embedding：text-embedding-3-small / bge-m3
- 向量库：Milvus 或 Qdrant
- Redis（对话上下文缓存）
- WebFlux（SSE 流式响应）

【核心架构】

```
用户提问
   ↓
NLU 意图识别（小模型 + 规则）
   ↓
RAG 检索（元数据 / 决策追溯库 / 历史对话）
   ↓
LLM Plan（确定调用哪些 Function）
   ↓
Function Calling 工具集
   ↓
权限校验 + 数据脱敏
   ↓
LLM 整合 → 自然语言 + 结构化卡片 + 操作按钮
   ↓
SSE 流式输出
   ↓
反馈采集（👍/👎）→ 学习层
```

【Function 工具集（必须实现）】

| Function | 用途 | 权限 |
|---|---|---|
| get_metric | 查询指标 | 按指标权限 |
| get_metric_breakdown | 维度拆解 | 同上 |
| get_anomaly | 异动列表 | — |
| attribute_anomaly | 归因分析 | — |
| get_user_profile | 用户画像 | 按用户权限 |
| get_org_profile | 企业画像 | 按企业权限 |
| query_segment | 查询/创建分群 | 分群权限 |
| trigger_playbook | 启动 Playbook | 二次确认 |
| create_journey | 创建 Journey | 二次确认 |
| start_experiment | 启动实验 | 二次确认 |
| register_decision | 登记决策追溯 | 必须 |
| create_ticket | 创建工单 | 二次确认 |
| assign_task | 派发任务 | 二次确认 |
| get_related_decisions | 检索历史相似决策 | 公开决策 |
| generate_report | 生成报表 | 报表权限 |

每个 Function 接口：

```java
public interface AssistantTool {
    String getName();
    String getDescription();
    JsonNode getParameterSchema();
    ToolResult execute(JsonNode args, UserContext userContext);
}
```

【对话流程实现】

```java
@Service
public class AssistantService {
    public Flux<AssistantMessage> chat(ChatRequest request) {
        UserContext ctx = SecurityContext.current();
        ConversationContext conv = getOrCreateConversation(request.getConversationId());
        
        // 1. 添加用户消息到上下文
        conv.addUserMessage(request.getMessage());
        
        // 2. RAG 检索
        List<Document> ragDocs = ragService.search(request.getMessage(), ctx);
        
        // 3. 构造 Prompt（系统提示 + RAG + 历史 + 用户消息）
        Prompt prompt = buildPrompt(systemPrompt, ragDocs, conv.getHistory(), request.getMessage());
        
        // 4. 流式调用 LLM with Function Calling
        return chatModel.streamCall(prompt, tools)
            .flatMap(chunk -> {
                if (chunk.isToolCall()) {
                    return executeToolWithPermission(chunk.getToolCall(), ctx)
                        .flatMapMany(result -> continueWithToolResult(prompt, result));
                } else {
                    return Mono.just(chunk.toMessage());
                }
            })
            .doOnNext(msg -> conv.addAssistantMessage(msg))
            .doOnComplete(() -> auditLog.record(conv));
    }
}
```

【权限校验】

每个 Function 调用前：

```java
public ToolResult executeToolWithPermission(ToolCall call, UserContext ctx) {
    Tool tool = toolRegistry.get(call.getName());
    
    // 1. 用户是否有权调用此 Function
    if (!permissionService.canUseTool(ctx, tool.getName())) {
        return ToolResult.error("您暂无权限执行此操作");
    }
    
    // 2. Function 参数中的数据是否在权限范围
    if (!permissionService.canAccessData(ctx, call.getArgs())) {
        return ToolResult.error("您暂无该数据权限");
    }
    
    // 3. 写操作必须二次确认
    if (tool.isWriteOperation() && !call.isConfirmed()) {
        return ToolResult.requireConfirmation(tool, call.getArgs());
    }
    
    return tool.execute(call.getArgs(), ctx);
}
```

【RAG 检索】

向量库存储：
- 指标元数据（指标定义、口径、关联 Playbook）
- 决策追溯库（决策记录）
- 历史高质量对话
- 业务文档（产品 wiki、PRD）

检索流程：
1. 用户提问 Embedding
2. Milvus 检索 Top K（默认 10）
3. 重排（rerank）
4. 注入 Prompt

【系统 Prompt 模板】

```
你是 yourcompany 的决策助理。

【你的能力】
- 帮用户查询业务指标
- 分析数据异动
- 给出决策建议
- 触发分群、Journey、实验、决策登记等操作

【你的原则】
- 永远输出"数据 + 解释 + 行动"三件套
- 每个数字标注 📎 数据来源
- 建议必须显示置信度
- 涉及操作必须用户二次确认
- 不替代人决策

【当前用户】
{userContext}

【相关历史对话】
{conversationHistory}

【RAG 检索到的相关信息】
{ragContext}

【可调用的工具】
{availableTools}

【用户当前消息】
{userMessage}
```

【6 级能力实现优先级】

- MVP：L1（查询）+ L2（趋势对比）+ L3（归因）
- V1：+ L4（行动建议）
- V2：+ L5（决策辅助）
- V3：+ L6（主动洞察）

【30 个场景脚本】

参考 docs/07-ai-assistant/02-conversation-scenarios.md。
每个场景应通过单元测试（mock LLM）验证。

【API】

```
POST /api/assistant/chat            # 对话（SSE 流式）
GET  /api/assistant/conversations   # 用户对话列表
GET  /api/assistant/conversations/{id}/messages
POST /api/assistant/feedback        # 👍 / 👎 反馈
POST /api/assistant/tool/confirm    # 二次确认
DELETE /api/assistant/conversations/{id}
```

【性能】

- 流式首字 < 800ms
- 简单查询响应 < 2s
- 复杂分析 < 8s
- 单用户 QPS 限 5/s
- 单租户 QPS 限 100/s

【安全】

- Prompt 注入防护：用户输入与系统提示词严格分离
- 所有 Function 权限校验
- 输出过滤敏感词
- 越权数据 → "您暂无该数据权限"
- LLM 幻觉防护：数据必带 SQL 来源
- 对话全留痕（脱敏后存 90 天）

【模型路由】

```
意图识别 → 小模型
归因分析 → 中型模型
决策辅助 → 大模型
工具调用 Plan → 中型模型
```

【验证】

1. 30 个场景脚本可正常运行
2. 权限隔离测试
3. Prompt 注入测试
4. Function 调用准确率
5. 用户 👍 率 ≥ 70%
6. 高峰流量下不雪崩

【交付物】

- 完整服务代码
- 15+ Function 实现
- Prompt 模板库
- RAG 配置
- 模型路由策略
- 评估测试集（30 场景）
- 监控指标

【禁止】

- 不允许 LLM 替代人决策
- 不允许 LLM 直接执行写操作（必须二次确认）
- 不允许跳过权限校验
- 不允许 AppSecret / 完整 SQL 出现在用户回答中

【强烈推荐】

- 用 Spring AI 或 LangChain4j 减少胶水代码
- 用 SSE 而非 WebSocket（更稳定）
- 用 Tool Calling 而非自实现 Plan
- 用 RAG 而非全部塞进 Prompt（成本）
- 用 Caffeine 缓存高频 Function 结果

现在请生成。
```
