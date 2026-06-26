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
## Day 6(周末):整合 + 单测 + Swagger ✅
- [[Mockito 单元测试]] —— @Mock 假依赖 + @InjectMocks 真被测对象，隔离测试 Service 层
- [[MockMvc 控制器测试]] —— @WebMvcTest 轻量容器 + MockMvc 模拟 HTTP，不启动服务器测 Controller
- [[Swagger 与 OpenAPI]] —— springdoc-openapi 自动生成交互式 API 文档，@Tag + @Operation 注解
- 更新:[[JUnit5]]（补 Mockito 配合使用场景）
- 代码：NoteServiceTest（6 个测试）、NoteControllerTest（6 个测试）、OpenApiConfig
- 踩坑：Spring Boot 4.x `@MockBean` → `@MockitoBean`，`@WebMvcTest` 包路径变更
- 自测加权 **55 分**（Q1=10 Q2=10 Q3=15 Q4=10 Q5=10），Day5 的 90 → Day6 的 55
- Q1 纠偏：@WebMvcTest 会加载轻量 Spring 容器（只装 Controller），不是完全不加载
- Q2 纠偏：@InjectMocks 自动扫描构造函数参数，找到匹配的 @Mock 注入
- Q5 纠偏：Swagger 关键注解 @Tag（分组）+ @Operation（接口描述）
## Day 7(周末):重构 + 复盘 + 双链 ✅
- [[全局异常处理]] —— 重构：RuntimeException → BusinessException（携带 HttpStatus）
- [[Java 异常体系]] —— 自定义业务异常继承 RuntimeException，精确区分业务异常 vs 代码 bug
- [[MOC 与双链]] —— Obsidian 知识管理方法论：MOC 枢纽笔记 + 双向链接构建知识网络
- 创建：[[Week2 MOC]] —— MOC 内容地图，用双链串联整周知识
- 双链更新：全局异常处理、三层架构、DI、Bean Validation、ApiResponse、Mockito、MockMvc、JUnit5、Java 异常体系（9 篇笔记新增关联）
- 代码：BusinessException.java、GlobalExceptionHandler 精确匹配、HelloController/GreetingService 清理
- 自测加权 **86 分**（Q1=95 Q2=90 Q3=85 Q4=100 Q5=60），Day6 的 55 → Day7 的 86
- Q2 纠偏：GlobalExceptionHandler 匹配顺序 — MethodArgumentNotValidException > BusinessException > Exception
- Q3 纠偏：MOC 核心是"双链"——正向链接（MOC→笔记）+ 反向链接（笔记→MOC）
- Q5 纠偏：Week 2 全景应包含三层架构 + 测试对应关系图，不能只列技术名词
- 里程碑：Week 2 全部达成 ✅（52 篇笔记，知识网络成型）

## 本周里程碑(目标)
- [x] `mvn spring-boot:run` 一键起,监听 8080
- [x] `/api/notes` 五个端点(POST / GET 列表 / GET 单个 / PUT / DELETE)跑通
- [x] `mvn test` 全绿（13 个测试：6 Service + 6 Controller + 1 上下文加载）
- [x] 全局异常 + 统一响应体

## 导航
- 上位:[[00-Map/总图谱]]
- 上一周:[[Week1 索引]]
- 下一周:[[Week3 索引]](待建)
