# Spring AI Advisors 拦截链

> Advisor 是 ChatClient 的"拦截器/中间件"——在 LLM 调用前后插入横切逻辑，不侵入业务代码。基于责任链模式实现。

## 它解决什么问题

Day 1-4 每次调 LLM 都是一条直线：`Controller → ChatClient.prompt() → LLM`。真实生产环境需要在调用前后做日志、权限、限流、缓存、指标采集。如果把这些逻辑写在业务代码里，每个 `prompt()` 调用都要重复一遍。

Advisor 把这些横切关注点抽成可插拔组件，挂载到调用链上：

```
没有 Advisor：
  ChatClient.prompt().user(msg).call() → LLM

有 Advisor：
  ChatClient.prompt()
      .advisors(a -> a
          .advisors(new SimpleLoggerAdvisor())  // 日志
          .advisors(new MetricsAdvisor()))      // 指标
      .user(msg)
      .call()
      → [日志 Advisor → 指标 Advisor → LLM]
```

类比：Express.js 的 `app.use(middleware)`、Python Flask 的 `@app.before_request`、Spring MVC 的 `HandlerInterceptor`。

## 核心要点

### 1. Advisor 接口层级

```
Advisor (顶层接口)
  ├── CallAdvisor        → 同步调用（call）
  │      adviseCall(request, chain) → response
  ├── StreamAdvisor      → 流式调用（stream）
  │      adviseStream(request, chain) → Flux<response>
  ├── MemoryAdvisor      → 对话记忆管理
  └── ToolAdvisor        → 工具调用（Function Calling）
```

### 2. SimpleLoggerAdvisor

Spring AI 内置最简单的 Advisor，在 DEBUG 日志中输出 request/response：

```java
// 默认构造：输出 request.toString() + response JSON
new SimpleLoggerAdvisor()

// Builder 定制
SimpleLoggerAdvisor.builder()
    .order(100)                     // 执行顺序
    .requestToString(r -> "...")    // 自定义 request 日志格式
    .responseToString(r -> "...")   // 自定义 response 日志格式
    .build()
```

注意：默认只在 **DEBUG** 级别输出。开启方式：
```xml
<!-- logback-spring.xml -->
<logger name="org.springframework.ai.chat.client.advisor.SimpleLoggerAdvisor"
        level="DEBUG"/>
```

### 3. 自定义 Advisor（Day 5 核心产出）

```java
public class MetricsAdvisor implements CallAdvisor {

    private final MeterRegistry meterRegistry;

    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request, CallAdvisorChain chain) {

        long start = System.nanoTime();

        // 调用下游（下一个 Advisor，或最终 LLM）
        ChatClientResponse response = chain.nextCall(request);

        long durationMs = TimeUnit.NANOSECONDS.toMillis(
                System.nanoTime() - start);

        // 提取 Token 用量
        var usage = response.chatResponse()
                .getMetadata().getUsage();

        // 上报 Micrometer 指标
        Timer.builder("llm.call.duration")
                .tag("model", "qwen-turbo")
                .register(meterRegistry)
                .record(durationMs, TimeUnit.MILLISECONDS);

        return response;
    }

    @Override public String getName() { return "metrics"; }
    @Override public int getOrder() { return 0; }
}
```

### 4. 挂载方式

```java
// 方式 1：per-request 挂载（只影响本次调用）
chatClient.prompt()
    .advisors(a -> a.advisors(
        new SimpleLoggerAdvisor(),
        new MetricsAdvisor(meterRegistry)))
    .user(text)
    .call()
    .content();

// 方式 2：全局默认 Advisor（影响所有调用）
@Bean
public ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultAdvisors(a -> a.advisors(new SimpleLoggerAdvisor()))
        .build();
}
```

### 5. 执行顺序

```
Advisor1 (order=0)  →  Advisor2 (order=1)  →  LLM
    ├─ before                    ├─ before
    │                           │
    │   chain.nextCall() ────────┘
    │                           │
    ├─ after                     ├─ after
```

`getOrder()` 越小越先执行。MemoryAdvisor 默认 `HIGHEST_PRECEDENCE + 200`。

## 面试怎么问

- **"ChatClient 的请求怎么拦截？"** → Advisor 接口，实现 CallAdvisor 或 StreamAdvisor，通过 `.advisors()` 挂载。责任链模式，每个 Advisor 决定是否调用 chain.nextCall()。

- **"SimpleLoggerAdvisor 和自定义 Advisor 有什么区别？"** → SimpleLoggerAdvisor 是 Spring AI 内置的调试工具，只写 DEBUG 日志。自定义 Advisor 能采集 Micrometer 指标、对接告警系统、做限流降级。

- **"多个 Advisor 的执行顺序怎么控制？"** → getOrder() 返回值，越小越外层（越早执行 before，越晚执行 after）。用 Ordered.HIGHEST_PRECEDENCE 常量计算相对偏移。

## 关联

- 使用场景：[[文档智能分析综合实战]]（Day 5 extractKeySentences 挂载 SimpleLoggerAdvisor）
- 前置：[[Spring AI 起步]]（ChatClient 基础）
- 类比：Express 中间件、Python decorator
- 进阶：[[结构化抽取完整链路]]（多 Advisor 链式组合）
- 可观测：Micrometer + Prometheus（Week 10 生产级加固）
