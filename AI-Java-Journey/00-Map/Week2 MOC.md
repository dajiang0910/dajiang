# Week 2 MOC：Spring Boot Web 开发

> MOC = Map of Content，内容地图。用双链串联一周所学，形成知识网络。

## 🗓️ 学习轨迹

| 天 | 主题 | 核心笔记 |
|---|---|---|
| Day 1 | Spring Boot 起步 + IoC + DI | [[Spring Boot 3 起手]] · [[Spring IoC 容器]] · [[依赖注入(DI)]] |
| Day 2 | REST API + DTO + 校验 | [[@RestController]] · [[DTO 与 Entity 之分]] · [[Bean Validation]] |
| Day 3 | JPA + H2 + 异常处理 | [[JPA 实体注解]] · [[H2 内存数据库]] · [[全局异常处理]] |
| Day 4 | MyBatis-Plus + 分页 | [[MyBatis-Plus 核心]] · [[分页查询（JPA vs MyBatis-Plus）]] |
| Day 5 | 统一响应 + 全局异常 | [[统一响应体（ApiResponse）]] · [[全局异常处理]] |
| Day 6 | 单元测试 + API 文档 | [[Mockito 单元测试]] · [[MockMvc 控制器测试]] · [[Swagger 与 OpenAPI]] |
| Day 7 | 重构 + 复盘 + 双链 | [[Java 异常体系]] · [[MOC 与双链]] · [[Week2 MOC]] |

## 🏗️ 三层架构（贯穿整周）

```
[[@RestController]]          ← Day 2 起
       ↓ 调用
[[依赖注入(DI)]]             ← Day 1 学
       ↓ 注入
[[@Service]]                 ← Day 1 起
       ↓ 调用
[[Spring Data JPA 与 JpaRepository]]  ← Day 3
[[MyBatis-Plus 核心]]        ← Day 4
```

## 🧪 测试体系（Day 6 建立）

```
[[MockMvc 控制器测试]]       ← 测 Controller（@WebMvcTest + MockMvc）
[[Mockito 单元测试]]         ← 测 Service（@Mock + @InjectMocks）
[[JUnit5]]                   ← 测试框架基础
```

## 🔗 关键概念网络

| 概念 | 连接到 |
|---|---|
| [[依赖注入(DI)]] | → [[构造器注入 vs 字段注入]] → [[面向接口编程]] |
| [[全局异常处理]] | ← [[Bean Validation]] → [[统一响应体（ApiResponse）]] |
| [[DTO 与 Entity 之分]] | ← [[JPA 实体注解]] → [[@RestController]] |
| [[application.properties 配置]] | → [[H2 内存数据库]] · [[MyBatis-Plus 核心]] |

## 🎯 里程碑检查

- [x] `mvn spring-boot:run` 启动成功
- [x] 5 个 REST 端点可用（GET/POST/PUT/DELETE）
- [x] `mvn test` 全绿（13 个测试）
- [x] 全局异常处理 + 统一响应体
- [x] Swagger UI 可访问

## 📊 知识资产

- Spring 笔记：25 篇（20-Spring/）（+1 MOC 与双链）
- Java 笔记：22 篇（10-Java/）
- 踩坑笔记：2 篇（30-踩坑/）
- Map 笔记：4 篇（00-Map/）
- **总计：53 篇**

## ➡️ 下一站

Week 3：数据库进阶 + MyBatis-Plus 深入 + 缓存
