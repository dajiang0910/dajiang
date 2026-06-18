# WeatherService 设计模式(I/O 与业务分离)

> CLI/Web/调度 上层只做 I/O,业务下沉到 Service 单测可达。Day 6 Week 1 收官的核心架构决策。

## 它解决什么问题
把所有逻辑堆在 `main()` 里(Week 1 Day 5 之前的状态)导致:
- **没法单测** —— 单测框架调不了 `main()`
- **没法换入口** —— 想从 Web 调同一个逻辑?复制粘贴
- **耦合死锁** —— 改打印格式要动业务代码

## 核心要点
**分层架构(三明治)**:
```
┌─────────────────────────────┐
│  I/O 层(Main)               │  ← 读命令行 / 读 HTTP 请求 / 读调度参数
│  - 解析输入                  │     打印结果 / 返回 HTTP 响应
│  - 调用 Service              │
│  - 格式化输出                │
└─────────────────────────────┘
              ↓
┌─────────────────────────────┐
│  业务层(Service)            │  ← 纯逻辑:URL 拼装 + HTTP + 解析
│  - 接收原始入参              │     单测只测这里(无 I/O)
│  - 返回业务对象              │
│  - 抛业务异常                │
└─────────────────────────────┘
              ↓
┌─────────────────────────────┐
│  数据层(外部 API)           │  ← Open-Meteo / 未来 LLM API
└─────────────────────────────┘
```

**好处**:
- **Service 单测可达**:不依赖命令行,不依赖网络,跑得快
- **入口可换**:同 Service 接 Web(REST controller)/CLI(picocli)/调度(定时任务)都共用
- **异常策略解耦**:Service 抛异常 → 上层决定打日志 / 返回 500 / 退出码非 0

## Java 里怎么落地
Day 6 三件套:
- `Main.java`(I/O 层):picocli 解析 `--city` → 调 Service → 打印
- `WeatherService.java`(业务层):`fetch(city)` 拼 URL + 发请求 + 解析
- `WeatherServiceTest.java`(单测):只测 `fetch` 参数校验,不调真实网络

**关键代码**:
```java
// Service 抛异常,不自己处理
public Weather fetch(String city) {
    CityParam.requireValid(city);
    // ... HTTP + 解析 ...
    if (response.statusCode() / 100 != 2) {
        throw new RuntimeException("HTTP " + response.statusCode());
    }
    return new Weather(...);
}

// Main 不 catch,异常冒到 picocli
@Override public void run() {
    Weather w = service.fetch(city);
    System.out.println("温度:" + w.temperature() + "°C");
}
```

## Week 2 预告
这模式**完全平移到 Spring**:
- I/O 层 → `@RestController`(读 HTTP 请求)
- 业务层 → `@Service`(纯业务)
- 单测 → `@SpringBootTest`(Week 8 再升级,加 `@MockBean`)

## 关联
- 测试:[[JUnit5]]
- CLI 入口:[[picocli]]
- 数据载体:[[record]]
- 静态检查:[[var 与静态类型]]