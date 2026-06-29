# 依赖注入(DI)

> DI(Dependency Injection)= IoC 的落地手段:容器在创建一个 Bean 时,自动把它依赖的其他 Bean「注入」进来,你不用自己 `new`。

## 它解决什么问题
让对象之间「解耦」:`HelloController` 不关心 `GreetingService` 怎么造、谁造的,只声明「我要一个」,容器负责给。换实现、写测试时只需换注入的 Bean,不改业务代码。

## 核心要点
- **三种注入方式**:
  - **构造器注入**(本周一律用这个 ✅):依赖通过构造器参数传入,可声明 `final`,不可变、不会注入到一半、便于单测。
  - Setter 注入:通过 setter 方法,适合可选依赖。
  - 字段注入(`@Autowired` 直接标字段):写着省事,但**不推荐**(不能 final、单测难、隐藏依赖)。
- **为什么用构造器注入**:① 依赖是 `final` 不可变 ② 依赖缺失启动就报错(快速失败)③ 单测可直接 `new Controller(mockService)`,不依赖 Spring 容器。
- **自己的类 vs 第三方类(关键分界线)**:
  - **你自己写的类** → 加 `@Component/@Service`,让容器扫描。
  - **JDK / 第三方的类(改不了源码)** → 在 `@Configuration` 类里用 `@Bean` 方法手动造。

## Java 里怎么落地
本项目构造器注入:
```java
@RestController
public class HelloController {
    private final GreetingService greetingService;            // final 不可变
    public HelloController(GreetingService greetingService) { // 容器自动注入
        this.greetingService = greetingService;
    }
}
```
第三方类(`HttpClient`/`ObjectMapper`)注册成 Bean —— Week1 的 `WeatherService` 搬进 Spring 时要这么干:
```java
@Configuration
public class AppConfig {
    @Bean ObjectMapper objectMapper() { return new ObjectMapper(); }
    @Bean HttpClient httpClient()     { return HttpClient.newHttpClient(); }
}
```

## 面试怎么问
- "IoC 和 DI 关系?" → IoC 是思想,DI 是实现手段。
- "字段注入 / 构造器注入选哪个?为什么?" → 构造器注入:final、快速失败、可测试。
- "`@Bean` 和 `@Component` 区别?" → `@Component` 标在自己的类上由容器扫描;`@Bean` 标在 `@Configuration` 的方法上,用于注册第三方类或需要自定义构造逻辑的对象。

## 关联
- 上位思想:[[Spring IoC 容器]]
- 工程入口:[[Spring Boot 3 起手]]
- 应用:[[@RestController]]
- 深入对比:[[构造器注入 vs 字段注入]](面试高频)
- 面向接口:[[面向接口编程]](DI 的真正威力是可替换性)
- 测试中的 DI：[[Mockito 单元测试]] 的 `@InjectMocks` 模拟构造器注入 · [[MockMvc 控制器测试]] 的 `@MockitoBean` 替换容器中的 Bean
- LLM 依赖注入：[[Spring AI 起步]] —— `ChatClient.Builder` 构造器注入，和 JpaRepository 一模一样
- Day 2 进阶：[[System 角色与消息类型]] · [[PromptTemplate 模板化提示词]] —— 同一个 ChatClient.Builder 支撑翻译、摘要、Slug 生成三个新方法
- Day 3 进阶：[[SSE 流式对话]] —— 同一个 ChatClient.Builder 同时支撑 `.call()`（同步）和 `.stream()`（流式），注入方式不变
- Day 4 进阶：[[多轮对话（消息历史）]] —— 新增 ChatMemory 注入（零配置，ChatMemoryAutoConfiguration 自动创建），构造器参数 +1
- Day 5 进阶：[[超时重试与Token成本]] —— 无新增注入依赖，重试通过 `SpringAiRetryAutoConfiguration` 自动配置（又是零代码！），Token 用量通过 `ChatClientResponse` API 获取
- Day 6 进阶：[[综合实战（智能笔记助手）]] —— 无新增注入依赖，`smartNote()` 复用同一个 ChatClient 实例执行 3 步调用链，Token 聚合依赖 Day 5 的 `ChatClientResponse` API
- Week 4 进阶：[[BeanOutputConverter 结构化输出]] —— `StructuredExtractService` 同样用 `ChatClient.Builder` 构造器注入，`.entity(Class)` 扩展了 ChatClient 的取值方式
