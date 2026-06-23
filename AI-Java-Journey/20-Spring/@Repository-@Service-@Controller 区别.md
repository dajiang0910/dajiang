# @Repository / @Service / @Controller 区别

> `@Controller`、`@Service`、`@Repository`、`@Component` 四个注解,在"让 Spring 扫描成 Bean"这件事上**没有本质区别**——底层都是 `@Component` 的派生注解。它们分四个,纯粹是为了**可读性和分层语义**。

## 底层真相

```java
@Target(ElementType.TYPE)
@Component                         // ← @Controller 本质就是 @Component
public @interface Controller { ... }

@Target(ElementType.TYPE)
@Component                         // ← @Service 也是
public @interface Service { ... }

@Target(ElementType.TYPE)
@Component                         // ← @Repository 也是
public @interface Repository { ... }
```

`@ComponentScan` 扫描时,四个注解都会被识别为 Bean 注册进容器,行为完全一样。

## 那为什么要分四个?

| 注解 | 语义 | 标在谁身上 |
|---|---|---|
| `@Component` | 通用组件 | 不属于下面三类的工具类 |
| `@Controller` | Web 控制层 | 处理 HTTP 请求的类 |
| `@Service` | 业务逻辑层 | 编排业务规则的类 |
| `@Repository` | 数据访问层 | 存取数据的类 |

**好处**:
- 看到 `@Service` 就知道这个类在业务层,不用翻代码
- IDE 和工具可以按注解做分层分析(依赖关系图、包扫描范围)
- `@Repository` 还有额外加持:Spring 自动把底层异常(如 `SQLException`)转换成 `DataAccessException` 体系(**异常翻译**),`@Component` 没有这个

## 面试怎么问

- "`@Service` 和 `@Component` 区别?" → 底层没区别,`@Service` 是 `@Component` 的特化,为了语义化分层。
- "只用 `@Component` 行不行?" → 技术上行,但违反分层规范,代码可读性差,团队协作时别人看不出这个类属于哪层。
- "`@Repository` 有什么额外能力?" → 异常翻译:把 `SQLException` 等底层异常转成 Spring 的 `DataAccessException`。

## 关联

- 分层架构:[[三层架构(Controller-Service-Repository)]]
- 注入方式:[[依赖注入(DI)]]
- Web 入口:[[@RestController]]
