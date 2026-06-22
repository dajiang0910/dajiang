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

## Day 2:三层架构 + 内存版 Repository(待学)
## Day 3:REST CRUD + Bean Validation + 统一响应体(待学)
## Day 4:全局异常处理 + 请求日志 + application.yml 多环境(待学)
## Day 5:MyBatis-Plus + H2/MySQL + 分页(待学)
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
