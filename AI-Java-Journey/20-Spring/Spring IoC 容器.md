# Spring IoC 容器

> IoC(控制反转)= 把「创建对象、管理依赖」的控制权从你手里交给 Spring 容器;容器是个「对象管家」,统一造好、连好、按需递给你。

## 它解决什么问题
Week1 你在 `Main` 里手动 `new WeatherService(client, mapper)`,对象一多,组装代码就成了一坨「谁 new 谁、谁依赖谁」的硬编码,改一处动全身。
IoC 把这件事反过来:你只声明「我是个 Bean / 我需要谁」,容器负责造和连。

## 核心要点
- **Bean**:被容器管理的对象。贴 `@Component`/`@Service`/`@RestController` 等标签的类,启动时被扫描、实例化、放进容器。
- **容器扫描注册三步**:启动 → 扫描 `@Component` 家族 → 实例化放进容器(注册成 Bean)。
- **默认单例(singleton)**:整个容器一个实例,到处注入的是同一个对象(省内存、无状态共享)。
- **IoC vs DI**:IoC 是**思想**(控制权反转给容器),DI(依赖注入)是 IoC 的**实现手段**。详见 [[依赖注入(DI)]]。
- **ApplicationContext**:容器的代表对象,`run()` 返回它;`ctx.getBeanDefinitionCount()` 看装了多少 Bean。

## Java 里怎么落地
Day1 实验:删掉 `GreetingService` 的 `@Service` → 启动报 `required a bean ... that could not be found` → 证明「没注册成 Bean 就注入不进来」。
```java
var ctx = SpringApplication.run(NotesApiApplication.class, args);
System.out.println(ctx.getBeanDefinitionCount());       // 一两百个(大多是自动配置的)
System.out.println(ctx.containsBean("greetingService")); // true
```

## 面试怎么问
- "什么是 IoC?" → 控制反转,把对象的创建和依赖管理交给容器。
- "Bean 默认作用域?" → 单例 singleton。
- "`@Component` / `@Service` / `@Repository` / `@Controller` 区别?" → 本质都是 Bean,语义不同(通用组件 / 业务层 / 数据层 / Web 层),方便分层与 AOP 切面定位。

## 关联
- 实现手段:[[依赖注入(DI)]]
- 工程入口:[[Spring Boot 3 起手]]
- 对照 Week1:[[WeatherService 设计模式]](原来手动 new,现在交给容器管)
