# SSE 流式对话

> `.stream()` 替代 `.call()`，返回 `Flux<String>`，LLM 回复边生成边推送，前端呈现"打字机"效果。

## 为什么需要流式

同步 `.call()` 的体验问题：

```
用户发消息 → 线程阻塞 3~30 秒 → 整段文字突然出现
             ↑ 用户干等，不知道"有没有在干"
```

流式 `.stream()` 的改善：

```
用户发消息 → "依" → "赖" → "注" → "入" → "是" → ...
             ↑ 每个字立刻出现，像 ChatGPT 打字机效果
```

**核心差异**：

| | `.call()` | `.stream()` |
|---|---|---|
| 返回类型 | `String` | `Flux<String>` |
| 阻塞行为 | 同步阻塞，等完整回复 | 异步非阻塞，边生成边推送 |
| 用户感知 | 干等 → 突然出现 | 逐 token 输出，打字机效果 |
| HTTP 协议 | 普通 JSON 响应 | SSE (`text/event-stream`) |
| 适用场景 | 翻译、摘要、短回复 | 对话、长篇生成、高并发 C 端 |

## `.stream()` 方法链

```java
// Day 1 — 同步
String reply = chatClient.prompt()
        .user(message)
        .call()          // ← 阻塞，返回 CallResponseSpec
        .content();      // ← String

// Day 3 — 流式
Flux<String> stream = chatClient.prompt()
        .user(message)
        .stream()        // ← 非阻塞，返回 StreamResponseSpec
        .content();      // ← Flux<String>
```

> 同一个 `ChatClient`，同一个方法链起点，换一个调用方法 → 完全不同返回类型和协议。

## Flux`<`String`>` 是什么

`Flux` 是 **Project Reactor** 的核心类型，代表 **0..N 个元素的异步序列**：

```
时间轴 →
LLM 生成:  "依" → "赖" → "注" → "入" → "是" → ...  → "[完成]"
             ↓      ↓      ↓      ↓      ↓
Flux:  ─────"依"───"赖"───"注"───"入"───"是"─── ... ──→ onComplete()
```

对比熟悉的类型：

| 类型 | 元素数 | 时序 |
|---|---|---|
| `String` | 1 个完整值 | 同步，一瞬间 |
| `List<String>` | N 个值，一次性 | 同步，一瞬间 |
| `Mono<String>` | 0..1 个值 | 异步，未来某时刻 |
| **`Flux<String>`** | **0..N 个值** | **异步，随时间逐个推送** |

> `Flux` 是 `reactor-core` 的类，通过 `spring-ai-client-chat` 传递依赖，无需额外引入。

## SSE（Server-Sent Events）

SSE 是 HTTP 协议之上的**服务器单向推送**机制：

```
HTTP 响应头:
Content-Type: text/event-stream    ← 告诉客户端 "这是事件流"
Cache-Control: no-cache
Connection: keep-alive

HTTP 响应体（持续追加，不关闭连接）:
data: 依赖

data: 注入

data: 是

data: 一种设计模式

...

data: [DONE]
```

**SSE vs WebSocket**：

| | SSE | WebSocket |
|---|---|---|
| 方向 | 单向（服务器→客户端） | 双向（全双工） |
| 协议 | HTTP（兼容所有 HTTP 设施） | 独立协议 `ws://` |
| 自动重连 | 内置 | 需手动实现 |
| 数据类型 | 文本 | 文本 + 二进制 |
| 复杂度 | 低 | 中 |
| 典型场景 | AI 流式输出、通知推送 | 实时聊天、协作编辑 |

> 💡 AI 对话流式输出用 SSE 是行业标准：OpenAI、Anthropic、阿里百炼的流式 API 全是 SSE。

## 代码实战

### Service 层

```java
@Service
public class ChatService {
    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    // Day 1 — 同步
    public String chat(String userMessage) {
        return chatClient.prompt().user(userMessage).call().content();
    }

    // Day 3 — 流式（新增）
    public Flux<String> streamChat(String userMessage) {
        return chatClient.prompt()
                .user(userMessage)
                .stream()       // .stream() 替代 .call()
                .content();     // 返回 Flux<String>
    }
}
```

### Controller 层

```java
// GET /api/chat/stream?message=什么是依赖注入
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamChat(@RequestParam String message) {
    return chatService.streamChat(message);
}
```

**关键点**：
- `produces = MediaType.TEXT_EVENT_STREAM_VALUE` 声明 SSE 格式（实际值 `text/event-stream`）
- 返回 `Flux<String>` 而非 `ApiResponse<String>`（流式不适合包装在统一响应体中）
- `@RequestParam` 通过 URL 查询参数传消息（适合 GET + EventSource）

### 测试方式

```bash
# curl -N 禁用缓冲，逐行输出
curl -N "http://localhost:8080/api/chat/stream?message=什么是依赖注入"
```

前端消费：
```javascript
// EventSource API（浏览器原生支持）
const es = new EventSource("/api/chat/stream?message=你好");
es.onmessage = (e) => {
    console.log(e.data);  // 每收到一个 token 打印一次
    // 实际应用：追加到聊天界面
};
```

## 三步走：call → stream 迁移

```
Day 1: .call()   → String        → 普通 HTTP JSON 响应
Day 3: .stream() → Flux<String>  → SSE (text/event-stream) 流式响应
       ↑
       同一个 ChatClient，同一个 .prompt()，只是调用方法不同
```

| 改动点 | 同步 | 流式 |
|---|---|---|
| ChatService 方法签名 | `String chat(...)` | `Flux<String> streamChat(...)` |
| ChatClient 调用 | `.call().content()` | `.stream().content()` |
| Controller 返回值 | `ApiResponse<String>` | `Flux<String>` |
| Content-Type | 默认 `application/json` | `produces = TEXT_EVENT_STREAM_VALUE` |
| 前端消费 | `fetch().then(res => res.json())` | `new EventSource(url)` |

## 面试怎么问

- "`.call()` 和 `.stream()` 的区别？" → `.call()` 同步返回 `String`，等完整回复；`.stream()` 异步返回 `Flux<String>`，边生成边推送
- "为什么 AI 对话要用流式？" → 用户体验（打字机效果）、减少感知等待、高并发下不长时间占用线程
- "SSE 和 WebSocket 选哪个？" → AI 流式输出用 SSE（单向推送、简单、HTTP 兼容）；需要双向实时通信用 WebSocket
- "`Flux` 是什么？从哪来的？" → Project Reactor 的 0..N 异步序列类型，通过 `spring-ai-client-chat` → `reactor-core` 传递依赖

## 关联

- 前置：[[Spring AI 起步]]（Day 1 — ChatClient 方法链基础）
- 前置：[[System 角色与消息类型]]（Day 2 — system/user 角色）
- 前置：[[PromptTemplate 模板化提示词]]（Day 2 — 模板化提示词）
- 注入支撑：[[依赖注入(DI)]]（同一个 ChatClient.Builder 支撑同步 + 流式）
- 端点注册：[[@RestController]]（新增 GET /api/chat/stream 端点）
- 响应对比：[[统一响应体（ApiResponse）]]（流式不包装 ApiResponse，直接返回 Flux）
- 后续：多轮对话（消息历史，Day 4）
