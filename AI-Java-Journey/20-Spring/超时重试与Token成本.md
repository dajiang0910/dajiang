# 超时重试与 Token 成本

> **LLM 调用不是"发请求就完事"**——要像管数据库连接池一样管它的超时、重试、和每一分钱。Day 5 为 ChatClient 套上"防护罩"。

## 它解决什么问题

没有超时/重试/成本意识时：
```java
// 裸调 LLM —— 网络卡 30 秒 → 线程池耗尽 → 服务雪崩
chatClient.prompt().user(msg).call().content();
// 429 限流 → 直接抛异常给用户
// Token 黑洞 → 月底账单吓一跳，攻击了也不知道
```

有了三层防护后：
```java
// 同一行代码，但底层多了：30s 超时 + 3 次指数退避重试 + Token 用量追踪
chatClient.prompt().user(msg).call().chatClientResponse();
// → 从 ChatResponse.getMetadata().getUsage() 获取 Token 用量
```

## Spring AI 的三层防护

```
┌─────────────────────────────────────────────────────────┐
│  1. 超时层（OpenAiChatOptions.timeout）                   │
│     30 秒不回复 → 直接掐断，不占用线程                      │
├─────────────────────────────────────────────────────────┤
│  2. 重试层（SpringAiRetryAutoConfiguration）              │
│     5xx / 网络异常 → 自动重试 3 次，指数退避（1s→2s→4s）    │
│     4xx 客户端错误 → 不重试（请求本身有问题，重试无意义）      │
├─────────────────────────────────────────────────────────┤
│  3. 成本层（ChatResponse.getMetadata().getUsage()）       │
│     每次调用记录输入/输出 Token，可算钱、可监控异常           │
└─────────────────────────────────────────────────────────┘
```

## 配置详解

### 超时配置

```properties
# LLM 调用最大等待时间（HTTP 读取超时）
spring.ai.openai.chat.options.timeout=30s
```

**为什么是 30 秒**：qwen-turbo 正常响应 2-5 秒，30 秒给了足够余量。生产环境可根据模型调整——大模型（qwen-max）设 60s，小模型设 15s。

**超时后会发生什么**：Spring AI 抛出 `TimeoutException`（或包装在 `RestClientException` 中），不会返回部分内容（和流式不同）。

### 重试配置

```properties
# 最多重试 3 次（不含首次调用，即总共最多 4 次尝试）
spring.ai.retry.max-attempts=3

# 退避策略（指数退避 Exponential Backoff）
spring.ai.retry.backoff.initial-interval=1000ms   # 第一次重试前等 1 秒
spring.ai.retry.backoff.multiplier=2               # 每次乘 2
spring.ai.retry.backoff.max-interval=10s           # 退避上限 10 秒

# 对 4xx 客户端错误不重试（请求本身有问题，重试也白费）
spring.ai.retry.on-client-errors=false
```

**重试时间线**（假设每次调用失败）：
```
首次调用（失败）
  ↓ 等 1s（initial-interval）
第 1 次重试（失败）
  ↓ 等 2s（1s × 2）
第 2 次重试（失败）
  ↓ 等 4s（2s × 2）
第 3 次重试（失败）
  ↓ 抛异常给调用方
  
总耗时 ≈ 调用耗时 × 4 + 7s（等待时间）
```

**重试 vs 不重试的判断法则**：

| HTTP 状态码 | 含义 | 是否重试 | 原因 |
|---|---|---|---|
| 429 | 限流 | ✅ 重试 | 等一会配额就恢复 |
| 500 | 服务器内部错误 | ✅ 重试 | 下次可能正常 |
| 502/503 | 网关错误/服务不可用 | ✅ 重试 | 瞬时故障 |
| 网络超时 | 连接/读取超时 | ✅ 重试 | 网络抖动 |
| 400 | 请求参数错误 | ❌ 不重试 | 重试也是 400 |
| 401 | 认证失败 | ❌ 不重试 | API Key 问题 |
| 404 | 模型不存在 | ❌ 不重试 | 配置问题 |

### Token 观测配置

```properties
# 调试时打开：在日志中看到发送的 prompt 和收到的 completion
spring.ai.chat.observations.log-prompt=false       # 生产建议关
spring.ai.chat.observations.log-completion=false   # 生产建议关
```

> ⚠️ 生产环境建议关闭：用户 prompt 和 AI 回复可能包含敏感信息（如内部文档内容），打全量日志有数据泄露风险。

## Token 用量获取

### 两条路径对比

| 路径 | API | 返回值 | 能拿 Token 用量？ | 使用场景 |
|---|---|---|---|---|
| 快捷路径 | `.call().content()` | `String` | ❌ 不能 | 不关心成本的简单场景 |
| 完整路径 | `.call().chatClientResponse()` | `ChatClientResponse` | ✅ 能 | 成本追踪、监控告警 |

### 调用链详解

```
.call()                          → CallResponseSpec    （同步调用入口）
  .chatClientResponse()          → ChatClientResponse  （包装对象）
    .chatResponse()              → ChatResponse        （模型响应）
      .getResult()               → Generation          （单条生成结果）
        .getOutput()             → AssistantMessage    （AI 回复消息）
          .getText()             → String              （纯文本 ✅）
      .getMetadata()             → ChatResponseMetadata（元数据）
        .getUsage()              → Usage               （Token 统计）
          .getPromptTokens()     → Integer             （输入 Token 数）
          .getCompletionTokens() → Integer             （输出 Token 数）
          .getTotalTokens()      → Integer             （总量 = 输入 + 输出）
        .getRateLimit()          → RateLimit           （限流信息）
          .getTokensRemaining()  → Long                （剩余 Token 配额）
          .getRequestsRemaining()→ Long                （剩余请求配额）
        .getModel()              → String              （实际使用的模型名）
        .getId()                 → String              （本次请求唯一 ID）
```

## 代码实战

### ChatService.chatWithCost()

```java
public ChatCostResponse chatWithCost(String userMessage) {
    var clientResponse = chatClient.prompt()
            .user(userMessage)
            .call()
            .chatClientResponse();  // ← 关键！不用 .content()

    var chatResponse = clientResponse.chatResponse();
    String reply = chatResponse.getResult().getOutput().getText();
    var usage = chatResponse.getMetadata().getUsage();

    return new ChatCostResponse(
            reply,
            new ChatCostResponse.TokenUsage(
                    usage.getPromptTokens(),
                    usage.getCompletionTokens(),
                    usage.getTotalTokens()
            )
    );
}
```

### ChatCostResponse DTO

```java
public record ChatCostResponse(
    String reply,
    TokenUsage tokens
) {
    public record TokenUsage(
        Integer inputTokens,      // 输入 Token 数
        Integer outputTokens,     // 输出 Token 数
        Integer totalTokens       // 总 Token 数
    ) {}
}
```

### Controller 端点

```java
@PostMapping("/chat/with-cost")
public ApiResponse<ChatCostResponse> chatWithCost(@Valid @RequestBody ChatRequest request) {
    return ApiResponse.success(chatService.chatWithCost(request.message()));
}
```

### 响应示例

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "reply": "Java 是一种面向对象的编程语言...",
    "tokens": {
      "inputTokens": 150,
      "outputTokens": 80,
      "totalTokens": 230
    }
  }
}
```

## 成本意识速算

### 百炼 qwen-turbo 定价（2025）

| 模型 | 输入（每 1K tokens） | 输出（每 1K tokens） | 典型 1 次对话成本 |
|---|---|---|---|
| qwen-turbo | ¥0.0005 | ¥0.001 | ~¥0.00045（~700 tokens） |

**直观感受**：
- 1 分钱 ≈ 22 次简单对话
- 1 元钱 ≈ 2200 次简单对话
- 如果你一天调 1 万次 → 约 ¥4.5

### Token 暴涨的告警信号

如果某次调用 Token 用量突然从 ~200 飙到 ~10000，可能意味着：
1. **Prompt 注入攻击**：用户塞了超长文本
2. **历史泄露**：ChatMemory 堆积了太多历史消息未清理
3. **Bug**：System Prompt 被意外重复拼接

**对策**：在 `chatWithCost()` 中加 Token 用量监控日志，超过阈值告警。

## 端点全景图（Day 1→6）

```
Day 1: POST /api/chat              → String（同步，无记忆，无成本）
Day 2: POST /api/chat/translate    → String（system 角色）
       POST /api/chat/summarize    → String（PromptTemplate）
       POST /api/chat/slug         → String（system + 模板）
Day 3: GET  /api/chat/stream       → Flux<String>（流式 SSE）
Day 4: POST /api/chat/multi-turn   → String（多轮，ChatMemory + Advisor）
Day 5: POST /api/chat/with-cost    → ChatCostResponse（同步 + Token 成本 ✅）
Day 6: POST /api/chat/smart-note   → SmartNoteResponse（多步调用链 + Token 聚合 🆕）
```

## 面试怎么问

- "LLM 调用怎么保证可靠性？" → 三重防护：超时掐断（防线程耗尽）、指数退避重试（防瞬时故障）、Token 监控（防成本异常）+ 区分 4xx（不重试）和 5xx（重试）
- "怎么知道每次调用花了多少钱？" → 用 `.chatClientResponse()` 拿 `Usage`：`getPromptTokens()` + `getCompletionTokens()`，对照模型定价算钱
- "超时和重试的配置怎么设？" → timeout 按模型响应时间设（qwen-turbo 30s，大模型 60s）；重试 3 次 + 指数退避，4xx 不重试
- "有哪些不能用默认值的配置？" → timeout（默认可能无限等待）、retry.backoff（默认固定间隔不如指数退避）、max-tokens（防止回复过长）
- "`spring.ai.retry.on-client-errors=false` 为什么？" → 4xx 是客户端请求错误（如 400 参数错、401 鉴权失败），重试也解决不了，反而浪费配额

## 关联

- 上级概念：[[Spring AI 起步]] —— ChatClient 是基础，超时/重试/成本是它的"防护罩"
- 底层机制：[[依赖注入(DI)]] —— `SpringAiRetryAutoConfiguration` 自动配置 `RetryTemplate`，零代码注入
- 端点注册：[[@RestController]] —— Day 5 新增 `POST /api/chat/with-cost`
- 响应包装：[[统一响应体（ApiResponse）]] —— 新端点复用 ApiResponse 包装 ChatCostResponse
- 历史管理：[[多轮对话（消息历史）]] —— ChatMemory 历史过长 → Token 暴涨的根源之一
- 流式对比：[[SSE 流式对话]] —— 流式调用也可以获取 Token 用量（通过 `Flux<ChatClientResponse>`）
- 生产进阶（W10）：Micrometer + Spring AI Observability → 自动采集 Token/延迟/调用链路
- Day 6 实战：[[综合实战（智能笔记助手）]] —— Token 聚合实战：3 步调用链每步用 `.chatClientResponse()` 获取 Usage，三步累加得链路总成本
