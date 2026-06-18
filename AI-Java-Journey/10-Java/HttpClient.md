# HttpClient

> JDK 自带的 HTTP 客户端(`java.net.http.HttpClient`),Java 11+ 用,类比 Python 的 httpx/requests。

## 它解决什么问题
调外部 API(LLM、向量库、第三方服务) —— W3 调 LLM、W4 调 RAG、W7 调 Agent 工具,全靠它。

## 核心要点
- 三件套:`HttpClient`(发送方)+ `HttpRequest`(请求)+ `HttpResponse<T>`(响应,T 由 BodyHandler 决定)
- 常用代码模式:
  ```java
  HttpClient client = HttpClient.newHttpClient();
  HttpRequest req = HttpRequest.newBuilder(URI.create(url))
      .timeout(Duration.ofSeconds(30))    // 必加,防卡死
      .header("Accept", "application/json")
      .GET().build();
  HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
  ```
- `send()` 抛 `IOException`、`InterruptedException`
- **send() 不查状态码** —— 401/429/500 都不自动抛,必须自己 `if (statusCode()/100 != 2)`

## 一个重要坑(面试/实战都重要)
`HttpResponse` 接口在 JDK 21 上**没有**实现 `AutoCloseable`,**javap -p 反编译可证**(见 [[踩坑-HttpResponse 不能 try-with-resources]])。同步 String 响应体读完不需要关;流式响应关的是 body 流本身。

## 实战必备
1. **加 timeout**:`HttpRequest` 必加 `.timeout()`,否则网络卡住一直等
2. **状态码检查**:非 2xx 抛异常 + 记 ERROR 日志
3. **重试**:429/500 用指数退避重试 2-3 次
4. **限流感知**:429 看 `Retry-After` 头
5. **熔断降级**:连续失败返回兜底答案

## 面试怎么问
- "HttpClient 怎么用?" → 三件套 + BodyHandler
- "HttpClient 状态码会自己抛异常吗?" → 不会,要自己查
- "HttpResponse 能 try-with-resources 吗?" → **不能**(JDK 21),见踩坑笔记

## 关联
- 踩坑:[[踩坑-HttpResponse 不能 try-with-resources]]
- 关流写法:[[try-with-resources]](同步 String 响应不需要,流式响应要)
- 配合:[[Jackson]] 解析响应
- 异常来源:[[Java 异常体系]](send() 抛 IOException/InterruptedException)
- 调试:[[SLF4J 日志]] 记请求/响应
