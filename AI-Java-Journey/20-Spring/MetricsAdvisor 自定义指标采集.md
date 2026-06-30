# MetricsAdvisor 自定义指标采集

> 自定义 CallAdvisor，在 LLM 调用前后采集耗时与 Token 用量到 Micrometer，比 SimpleLoggerAdvisor 从"调试工具"升级到"生产级可观测"。Week 4 Day 6 周末整合核心产出。

## 它解决什么问题

Day 5 引入了 SimpleLoggerAdvisor —— 但它只在 DEBUG 日志打 request/response，纯调试用途。真实生产环境需要回答：

- "LLM 调用 P95 延迟是多少？有没有突然变慢？"
- "今天花了多少 Token？哪个模型的调用量最大？"
- "调用失败率是多少？"

这些问题 SimpleLoggerAdvisor 回答不了——它只是 `log.debug()`。MetricsAdvisor 把每次 LLM 调用的耗时和 Token 用量上报到 Micrometer 指标注册中心，后续对接 Prometheus + Grafana 就能可视化、设告警。

**类比**：SimpleLoggerAdvisor 像 `console.log("request took 300ms")`，MetricsAdvisor 像 `statsd.timing("llm.duration", 300)`。

## 核心要点

### 1. CallAdvisor 接口

```java
// Spring AI 2.0 实际包路径
package org.springframework.ai.chat.client.advisor.api;

public interface CallAdvisor extends Advisor {
    ChatClientResponse adviseCall(
        ChatClientRequest request,      // prompt、messages、model
        CallAdvisorChain chain           // 责任链引用
    );
}
```

执行模板 = **before → chain.nextCall() → after**，跟 Express.js 中间件完全相同：

```java
public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
    // === before ===
    long start = System.nanoTime();

    // === 放行到下游（下一个 Advisor，或最终 LLM）===
    ChatClientResponse response = chain.nextCall(request);

    // === after ===
    long durationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    // 上报指标...

    return response;
}
```

### 2. 三个 Micrometer 指标

| 指标名 | 类型 | Tag | 用途 |
|--------|------|-----|------|
| `llm.call.duration` | Timer | model | 耗时分布（P50/P95/P99）+ 调用次数 |
| `llm.call.count` | Counter | model | 累计调用次数（算速率：`rate(llm.call.count[5m])`） |
| `llm.call.tokens` | Counter | model, type | 按 prompt/completion/total 分别累加 Token |

**为什么 duration 用 Timer？**
- Timer 自动计算 count + totalTime + max + P50/P75/P95/P99 分位数
- Gauge 只记录瞬时值，看不到分布
- 一个 Timer 替代了 Counter（次数）+ 分位数计算

**为什么 tokens 用 Counter？**
- Token 用量是单调递增的累计值，Counter 语义天然匹配
- `Counter.increment(n)` 只增不减
- 在 Prometheus 中用 `rate(llm.call.tokens[1h])` 看每小时 Token 消耗速率

### 3. 防御性编程

```java
try {
    // 提取 Token 用量...
} catch (Exception e) {
    log.warn("MetricsAdvisor 采集 Token 用量失败：{}", e.getMessage());
}
```

**核心原则：可观测是辅助，不能变故障源。** 指标采集失败不应影响 LLM 调用本身。

### 4. 执行顺序

```
MetricsAdvisor before (order=HIGHEST_PRECEDENCE, 开始计时)
  → SimpleLoggerAdvisor before (order=0, 打印 request 日志)
    → LLM
  → SimpleLoggerAdvisor after (打印 response 日志)
MetricsAdvisor after (停止计时, 上报 Micrometer)
```

`getOrder()` 返回 `Ordered.HIGHEST_PRECEDENCE`（最小 int 值），确保 MetricsAdvisor 在最外层——先开始计时，最后结束计时，这样测量的耗时包含链上所有 Advisor 的开销。

### 5. 挂载方式

```java
// per-request 挂载（推荐：灵活控制哪些调用被监控）
chatClient.prompt()
    .advisors(a -> a.advisors(
        new MetricsAdvisor(meterRegistry, "qwen-turbo"),
        new SimpleLoggerAdvisor()))
    .user(text)
    .call()
    .content();

// 全局挂载（慎用：所有调用都被监控）
@Bean
public ChatClient chatClient(ChatClient.Builder builder, MeterRegistry registry) {
    return builder
        .defaultAdvisors(a -> a.advisors(new MetricsAdvisor(registry)))
        .build();
}
```

**注意**：`advisors(a -> a.advisors(...))` 不是拼写错误——是 `a.advisors()`（复数），不是 `a.advisor()`（单数不存在）。

## Java 里怎么落地

完整实现见项目代码：
```java
// src/main/java/com/example/notes_api/advisor/MetricsAdvisor.java
public class MetricsAdvisor implements CallAdvisor {

    private final MeterRegistry meterRegistry;
    private final String defaultModel;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        long startNanos = System.nanoTime();
        ChatClientResponse response = chain.nextCall(request);
        long durationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNanos);

        Timer.builder("llm.call.duration").tag("model", defaultModel)
                .register(meterRegistry).record(durationMs, TimeUnit.MILLISECONDS);

        Counter.builder("llm.call.count").tag("model", defaultModel)
                .register(meterRegistry).increment();

        // Token 提取 + 上报（含 try-catch 防御）
        // ...

        return response;
    }
}
```

测试验证（7 个用例）：
```bash
mvn test -Dtest="MetricsAdvisorTest"
# 验证：调用次数、耗时、Token 用量、null 防御、累加、getName/getOrder
```

## 面试怎么问

- **"你们怎么监控 LLM 调用的？"** → 基于 Spring AI Advisor 接口自定义 MetricsAdvisor，在 adviseCall() 中采集耗时（Timer）+ Token（Counter）到 Micrometer，对接 Prometheus + Grafana 实现可视化和告警。指标采集用 try-catch 包裹，失败不影响业务。

- **"为什么不用 SimpleLoggerAdvisor 就够了？"** → SimpleLoggerAdvisor 是调试工具，只写 DEBUG 日志。生产环境需要量化指标：P95 延迟有无恶化、Token 消耗是否异常增长、调用失败率趋势——这些都依赖 Micrometer 指标而非文本日志。

- **"多个 Advisor 的执行顺序怎么控制？"** → getOrder() 返回值，越小越外层。MetricsAdvisor 用 Ordered.HIGHEST_PRECEDENCE 确保在最外层计时，SimpleLoggerAdvisor 用默认 order=0 在内层打日志。

- **"Timer 和 Counter 的区别？"** → Timer 记录每次事件的耗时并自动计算分位数，Counter 是单调递增的累计值。耗时用 Timer（需要 P95），Token 用量用 Counter（累计 + 可算 rate）。

## 关联

- 上游：[[Spring AI Advisors 拦截链]]（Day 5 SimpleLoggerAdvisor 入门）
- 使用场景：[[文档智能分析综合实战]]（extractKeySentences 挂载 Advisor）
- 指标基础设施：Micrometer + Prometheus + Grafana（Week 10 生产级加固）
- 对比参考：Express.js 中间件、Python Flask `@app.before_request`
- 测试：SimpleMeterRegistry（Micrometer 内存实现，不依赖外部服务）
