# 踩坑-HttpClient send() 状态码漏判

> Java 11+ `HttpClient.send()` 拿到响应后**不会自动检查状态码**,401/404/429/500 都被当作"正常响应"返回,需要自己手动判断。错过这个判断,程序会"成功"地把 HTML 错误页或异常 JSON 当成业务数据解析。

## 现象
原本应该抛"城市未找到"异常的代码,在城市名打错时反而返回了一坨 HTML,被 Jackson 解析成乱七八糟的字段,温度 0、风速 0,看着像"晴朗无风"——但其实是 Open-Meteo 返回的 404 错误页。

## 复现代码(删掉判断那行)
```java
// WeatherService.fetch(...) 里的关键代码
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
// ❌ 漏了: if (resp.statusCode() / 100 != 2) { throw ... }
JsonNode root = mapper.readTree(resp.body());  // 把 HTML 错误页当 JSON 解析
double temp = root.get("current").get("temperature_2m").asDouble();  // 返回 0
return new Weather(city, temp, 0, 0);
```

## 排查思路(3 步定位)
1. **打日志看 raw 响应**:`log.info("状态码:{}, body:{}", resp.statusCode(), resp.body())` —— 一眼看到 `404` + HTML 标签
2. **看业务字段异常**:温度 0、风速 0、weatherCode 0(晴朗默认值)→ "无风晴朗" 这种不合常理的结果 = 数据源头错了
3. **对照 [[踩坑-HttpResponse 不能 try-with-resources]] 一起看**:那个踩坑是"关流",这个踩坑是"判状态",都是"send() 拿到响应后你自己得管"

## 修复(3 行)
```java
HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
if (resp.statusCode() / 100 != 2) {    // 2xx 才算成功
    log.error("HTTP {} for city={}, body={}", resp.statusCode(), city, resp.body());
    throw new RuntimeException("天气服务异常: HTTP " + resp.statusCode());
}
```

## 为什么这个坑很危险
- **不报错**:`JsonNode.get("xxx")` 字段缺失返回 null,`.asDouble()` 返回 0,**没有任何异常**
- **不报警**:没 ERROR 日志,只有 INFO 级别的"请求成功"
- **难复现**:正常情况下 200 一切正常,只有传错参数才暴露
- **生产事故案例**:LLM 调用如果不做状态码判断,429 限流会被当成"模型没答" → 业务方以为 LLM 不灵 → 找错原因

## 完整防御(Week 1-2 应做到)
1. **状态码检查**(本踩坑)✅
2. **超时设置**:`HttpRequest.timeout(Duration.ofSeconds(10))` —— [[HttpClient]]
3. **重试**:429/500 用指数退避重试 2-3 次 —— 看 [[HttpClient]] 实战必备第 3 条
4. **熔断**:连续失败返回兜底答案 —— W10 学
5. **限流感知**:429 看 `Retry-After` 头 —— [[HttpClient]] 实战必备第 4 条

## 为什么回归测试很难写
- 真实 HTTP 难以稳定复现:CI 跑测试时不能保证 Open-Meteo 此刻是 404
- `HttpClient.send()` 是 `final` class,JDK 自带无法直接 mock
- 必须用 [[Mockito]] + MockWebServer,或抽接口 `WeatherApi` 包一层 → mock 接口(Week 8 学)

## 教训
> **Java HttpClient 是个"哑客户端":它不替你做协议层的判断,401/429/500 全要自己看。** 这是和 Python `requests` 最大的区别 —— `requests.raise_for_status()` 是默认开箱即用的。
>
> 同样的坑在 Week 3 接 LLM 时会再遇到一次:LLM 返回 429 限流、500 服务挂、空 body 都要自己处理。

## 关联
- 实战:[[HttpClient]]
- 异常策略:[[Java 异常体系]](状态码异常属于"外部世界不可控",走 checked 异常 + 重试)
- 架构:[[WeatherService 设计模式]](Service 抛异常、上层决定怎么应对)
- 进阶:Week 8 [[Mockito]] + MockWebServer 写稳定回归测试
