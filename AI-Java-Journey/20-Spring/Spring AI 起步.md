# Spring AI 起步

> Spring AI 把 LLM 变成后端的"普通依赖"——和 JPA 操作数据库一样：加依赖 → 配属性 → 注入 Bean → 调方法。

## 它解决什么问题

直接调 LLM API 要自己管 HTTP 连接池、拼 JSON、解析 SSE、处理重试。Spring AI 把这些封装成 Spring Bean，**LLM 能力像数据库一样被注入到任何需要的地方**。

## 核心理念：LLM 只是后端的普通依赖

| | JPA（数据库） | Spring AI（LLM） |
|---|---|---|
| 依赖 | `spring-boot-starter-data-jpa` | `spring-ai-starter-model-openai` |
| 注入 | `@Autowired NoteRepository` | `ChatClient.Builder` 构造器注入 |
| 调用 | `noteRepository.findById(id)` | `chatClient.prompt().user(msg).call().content()` |
| 配置 | `spring.datasource.url=...` | `spring.ai.openai.base-url=...` |

## Maven 依赖

```xml
<!-- Spring AI OpenAI Starter（一站式：模型 + 自动配置 + ChatClient + 观测） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
    <version>2.0.0</version>
</dependency>
```

**Spring AI 2.0 模块拆分**（1.x → 2.0 关键变化）：

| 1.x | 2.0 | 变化 |
|---|---|---|
| `spring-ai-openai-spring-boot-starter`（大单体） | `spring-ai-openai` | 纯模型实现，独立模块 |
| （塞在 starter 里） | `spring-ai-autoconfigure-model-openai` | 自动配置独立 |
| （塞在 starter 里） | `spring-ai-client-chat` | ChatClient API 独立 |
| — | `spring-ai-starter-model-openai` | 聚合 Starter，一键引入上面三个 |

> 💡 跟 Spring Boot 自己的模块化哲学一致：`spring-boot-starter-webmvc`（聚合）→ `spring-webmvc`（实现）+ `spring-boot-autoconfigure`（配置）。

## 阿里百炼配置

Spring AI 的 OpenAI 模块是**通用 OpenAI 协议客户端**，不锁定 OpenAI 官网。阿里百炼兼容此协议：

```properties
# API Key
spring.ai.openai.api-key=${DASHSCOPE_API_KEY:your-api-key-here}

# 百炼兼容 OpenAI 格式的端点（关键！）
spring.ai.openai.base-url=https://dashscope.aliyuncs.com/compatible-mode/v1

# 模型选择
spring.ai.openai.chat.options.model=qwen-turbo

# 温度 0.0~2.0
spring.ai.openai.chat.options.temperature=0.7
```

## ChatClient 方法链

```
chatClient.prompt()      → ChatClientRequestSpec  （准备请求）
    .system("角色设定")   → ChatClientRequestSpec  （系统消息，Day 2）
    .user("用户输入")     → ChatClientRequestSpec  （用户消息）
    .call()              → CallResponseSpec       （同步阻塞调用）
//  .stream()            → StreamResponseSpec     （流式调用，Day 3）
    .content();          → String                 （提取回复文本）
```

### ⚠️ 易错点：`.call()` 返回的是 `CallResponseSpec`，不是 `ChatClientResponse`

- `call()` → **`CallResponseSpec`**（接口，可继续链式调用）
- `CallResponseSpec.chatClientResponse()` → `ChatClientResponse`（取值方法）
- `CallResponseSpec.content()` → `String`（直接取文本，最常用）

## 代码实战

```java
// ChatService —— 注入 ChatClient.Builder，一行调 LLM
@Service
public class ChatService {
    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String chat(String userMessage) {
        return chatClient.prompt()
                .user(userMessage)
                .call()
                .content();
    }
}

// ChatRequest —— record 做请求体 DTO
public record ChatRequest(
        @NotBlank(message = "消息不能为空") String message
) {}

// ChatController —— 和 CRUD Controller 结构一模一样
@RestController
@RequestMapping("/api")
public class ChatController {
    private final ChatService chatService;

    public ChatController(ChatService chatService) {
        this.chatService = chatService;
    }

    @PostMapping("/chat")
    @Operation(summary = "同步对话")
    public ApiResponse<String> chat(@Valid @RequestBody ChatRequest request) {
        return ApiResponse.success(chatService.chat(request.message()));
    }
}
```

## 同步调用 `call()` 的特点

**时间线**：发起请求 → 线程全程阻塞等待 LLM 生成完整回复（~2-5 秒）→ 收到完整响应 → 线程继续

| 适合 | 不适合 |
|---|---|
| 短文本问答 | 长文本生成 |
| 后台批量处理 | 在线聊天流式交互 |
| 内部工具调用 | 高并发 C 端接口 |
| AI 润色/翻译 | 推理耗时不可控的复杂任务 |

## 关联

- 依赖注入：[[依赖注入(DI)]]（ChatClient.Builder 构造器注入，跟 JpaRepository 一样）
- 控制器结构：[[@RestController]]（ChatController 和 NoteController 三层结构完全一致）
- 请求体：[[DTO 与 Entity 之分]]（ChatRequest 用 record，不可变传输对象）
- 响应体：[[统一响应体（ApiResponse）]]（chat 端点复用同一套 ApiResponse 包装）
- 配置：[[application.properties 配置]]（新增 spring.ai.openai.* 配置段）
- 后续：[[System 角色与消息类型]]（Day 2 — system/user/assistant 三种角色）
- 后续：[[PromptTemplate 模板化提示词]]（Day 2 — 模板化提示词，命名占位符）
- 后续：[[SSE 流式对话]]（Day 3 — `.stream()` 替换 `.call()`，`Flux<String>` + `text/event-stream`）
- 后续：[[多轮对话（消息历史）]]（Day 4 — ChatMemory + MessageChatMemoryAdvisor，让 LLM "记住"上下文）
- 后续：[[超时重试与Token成本]]（Day 5 — 超时/重试/Token 成本，让 LLM 调用具备生产级可靠性）
- 后续：[[综合实战（智能笔记助手）]]（Day 6 — Day 1-5 全部能力整合为多步 AI 调用链）
- Week 4 进阶：[[BeanOutputConverter 结构化输出]]（Week 4 — `.entity(Class)` 让 LLM 返回类型安全的 Java Bean，ChatClient 的新取值方法）
