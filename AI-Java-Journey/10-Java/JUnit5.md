# JUnit5

> Java 主流单元测试框架(JUnit5 = JUnit Platform + Jupiter + Vintage)。Week 1 验收 `mvn test` 全绿靠它。

## 它解决什么问题
没单测 = "我跑通了" 不能变成"我的代码正确"。
单测断言"给定输入 → 期望输出",回归测试时一行 `mvn test` 全绿即代表没把老功能改坏。

## 核心要点
- **3 个核心注解**:
  - `@Test` —— 标注方法是测试方法
  - `@BeforeEach` —— 每个测试方法前跑(初始化)
  - `@AfterEach` —— 每个测试方法后跑(清理)
  - `@BeforeAll` / `@AfterAll` —— 整个测试类前后跑(必须 `static`)
- **3 个核心断言**:
  ```java
  assertEquals(expected, actual);                 // 期望等于实际
  assertThrows(IllegalArgumentException.class,    // 期望抛某异常
               () -> service.fetch(""));
  assertTrue(condition);                          // 条件为真
  ```
- **命名约定**:方法名 `场景_条件_期望`,如 `fetch_空字符串_抛IllegalArgumentException`,失败时一眼能看懂。
- **运行**:Maven 用 surefire 插件 + `mvn test` 跑 JUnit5。

## Maven 接入
```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>      <!-- test 范围:只测试时编译,不打进 jar -->
</dependency>
```
> Week 1 用 surefire 默认配置即可,Week 8 学 Mock 时再升级(那时用 Mockito + `@ExtendWith` 模拟 HTTP 响应)。

## Java 里怎么落地
Day 6 `WeatherServiceTest`:3 个测试覆盖"输入校验"边界(空字符串 / 纯空白 / null),跑得快、稳、不依赖网络。

## 面试怎么问
- "@Test 和 main 方法区别?" → @Test 由 JUnit 框架调用,失败不中断,统计结果。
- "为什么测试方法不能有依赖顺序?" → 测试必须独立,否则前一个失败后面全错。

## 关联
- 应用:[[WeatherService 设计模式]](业务层独立可单测)
- Week 8 会升级:[[Java 集合框架]]/[[HttpClient]]/[[Jackson]] 用 Mockito 模拟