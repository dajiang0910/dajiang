# Week 2 索引

> **Spring Boot 3 后端骨架**。本周目标:从 CLI 工具升级为分层清晰的 REST 服务,产出 `/api/notes` 完整 CRUD(校验 + 统一响应 + 全局异常 + 持久化)。工程独立建在 `notes-api/`,与 Week1 的 CLI 分开。

## Day 1:Spring Boot 工程开张 + DI/Bean ✅
- [[Spring Boot 3 起手]] —— start.spring.io 生成,`run()` 起内嵌 Tomcat
- [[Spring IoC 容器]] —— 容器管 Bean,默认单例,扫描注册
- [[依赖注入(DI)]] —— 构造器注入;自己的类 `@Component` / 第三方类 `@Bean`
- [[@RestController]] —— = `@Controller` + `@ResponseBody`,返回值直接转 JSON
- [[@SpringBootApplication]] —— 三注解组合(Day6 深挖自动配置)
- 还债:[[try-with-resources]] 补完 —— Suppressed 异常 + 实战 3 例 + 纠正 HttpResponse 错误
- 自测盲区:`@ResponseBody` 的作用机制(Q2,已纠偏入 [[@RestController]])

## Day 2:三层架构 + 内存版 Repository ✅
- [[三层架构(Controller-Service-Repository)]] —— Controller → Service → Repository,依赖只能从上往下
- [[面向接口编程]] —— Service 依赖接口不依赖实现,Day5 换 MySQL 只换实现类
- [[构造器注入 vs 字段注入]] —— final 不可变 + 快速失败 + 可测试 + 依赖显式(面试高频)
-[[@Repository-@Service-@Controller 区别]] —— 底层都是 @Component,分四个是为语义化分层
-[[@PostConstruct 与 Bean 生命周期]] —— 实例化 → 注入 → @PostConstruct → 就绪 → @PreDestroy
- [[DTO 与 Entity 之分]] —— Entity 用 class(ORM 要求可变),DTO 用 record(不可变传输)
- [[ConcurrentHashMap]] —— 线程安全 HashMap,CAS + synchronized 锁单个桶
- 自测加权 **94 分**(Q1=90 Q2=100 Q3=95 Q4=95 Q5=85),Day1 的 84 → Day2 的 94
- Q5 纠偏:HashMap 并发 put 丢数据 + JDK7 死循环(头插法链表成环),答得不够具体,已补充
## Day 3:REST CRUD + Bean Validation + 统一响应体 ✅
- [[Bean Validation]] —— @Valid / @NotBlank / @Size,校验失败抛 MethodArgumentNotValidException
- [[统一响应体（ApiResponse）]] —— code + message + data 泛型包装,前端统一拦截
- [[全局异常处理]] —— @RestControllerAdvice + @ExceptionHandler,拦截异常返回统一格式
- 更新:[[DTO 与 Entity 之分]]（补 Day3 实战）、[[@RestController]]（补 @RequestBody 详解）
- 代码：`NoteController` 完整 CRUD、`NoteService` 增删改查、`GlobalExceptionHandler`、`ApiResponse`
- 踩坑：Spring Boot 3.x 需单独引入 `spring-boot-starter-validation`（2.x 内置）
- 自测加权 **98 分**（Q1=90 Q2=100 Q3=100 Q4=100 Q5=100），Day2 的 94 → Day3 的 98
## Day 4:Spring Data JPA + 数据库持久化 ✅
- [[Spring Data JPA 与 JpaRepository]] —— 继承 JpaRepository 即拥有 CRUD，0 行实现代码
- [[JPA 实体注解]] —— @Entity / @Table / @Id / @GeneratedValue / @Column + 无参构造
- [[H2 内存数据库]] —— 免装 MySQL，H2 控制台直接看数据
- [[application.properties 配置]] —— 数据源 + JPA + H2 控制台 + ddl-auto 选项
- 更新:[[面向接口编程]]（补 Day 4 实战：JPA 替换内存实现，Service 零修改）
- 代码：NoteRepository 从 30 行实现类 → 1 行接口（extends JpaRepository），Note 加 JPA 注解
- 踩坑：Spring Boot 4.x 需单独引入 `spring-boot-h2console`（从 autoconfigure 拆出独立模块）
- 自测加权 **90 分**（Q1=100 Q2=100 Q3=100 Q4=100 Q5=50），Day3 的 98 → Day4 的 90
- Q5 纠偏：JPA 无参构造函数与反射机制（Hibernate 从数据库加载数据 → 反射创建空对象 → 逐字段填充）
## Day 5:MyBatis-Plus + 分页查询 ✅
- [[MyBatis-Plus 核心]] —— BaseMapper 继承即 CRUD，@TableName/@TableId/@TableField 注解
- [[分页查询（JPA vs MyBatis-Plus）]] —— JPA Pageable + Page`<`T`>` vs MP Page + IPage`<`T`>`，页码起点不同
- 更新: [[Spring Data JPA 与 JpaRepository]]（补 JPA vs MP 对比表）
- 代码：NoteMapper（extends BaseMapper）、MyBatisPlusConfig（分页拦截器 + SqlSessionFactory）、Controller 双端点对比
- 踩坑：① MP 3.5.9 PaginationInnerInterceptor 移到 mybatis-plus-jsqlparser 模块 ② JPA+MP 共存需手动配置 SqlSessionFactory
- 自测加权 **90 分**（Q1=90 Q2=100 Q3=100 Q4=100 Q5=60），Day4 的 90 → Day5 的 90
- Q5 纠偏：选 MP 理由不能只说"JPA 繁琐"，要讲 SQL 控制力 + 性能调优 + 可维护性
## Day 6(周末):整合 + 单测 + Swagger(待学)
## Day 7(周末):重构 + 复盘 + 双链(待学)

## 本周里程碑(目标)
- [ ] `mvn spring-boot:run` 一键起,监听 8080
- [ ] `/api/notes` 五个端点(POST / GET 列表 / GET 单个 / PUT / DELETE)跑通
- [ ] `mvn test` 全绿
- [ ] 全局异常 + 统一响应体

## 导航
- 上位:[[00-Map/总图谱]]
- 上一周:[[Week1 索引]]
- 下一周:[[Week3 索引]](待建)
