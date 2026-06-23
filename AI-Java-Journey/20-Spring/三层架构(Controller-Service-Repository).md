# 三层架构(Controller-Service-Repository)

> Java 后端的标准分层:Controller 只管收发 HTTP,Service 只管业务规则,Repository 只管数据存取。依赖**只能从上往下**,绝不反向、绝不跨层。

## 它解决什么问题

把所有逻辑塞进一个路由处理函数——能跑,但项目一大就烂:改数据库要动路由、写单测要启动 HTTP、业务逻辑无法复用。三层架构用**职责分离**把这三件事拆开,每层只管自己的事。

## 三层职责

```
HTTP 请求
   │
   ▼
┌─────────────┐
│ Controller  │  @RestController — 只管「收请求 / 参数校验 / 返响应」,不写业务逻辑
└──────┬──────┘
       │ 调用
       ▼
┌─────────────┐
│  Service    │  @Service — 只管「业务规则 / 编排」,不碰 HTTP、不写 SQL
└──────┬──────┘
       │ 调用
       ▼
┌─────────────┐
│ Repository  │  @Repository — 只管「存取数据」,今天内存,Day5 换数据库
└─────────────┘
```

## 铁律:依赖方向只能向下

- Controller → Service → Repository ✅
- Repository → Service ❌(反向依赖)
- Controller → Repository ❌(跨层调用)

**跨层调用的后果**:
- 破坏单一职责——Controller 开始写业务逻辑
- 业务无法复用——别的接口要用同样的查询,得重写一遍
- 单元测试困难——Controller 直接依赖 Repository,没法 mock Repository 来单独测 Controller

## 用别的语言类比

| 你以前的世界 | Java 三层 |
|---|---|
| Express router handler / Flask view | **Controller** |
| 你抽出来的 `service.js` / `services.py` | **Service** |
| 你的 `dao` / `models` / ORM 封装 | **Repository** |

差别:Java 用**注解 + 接口 + DI**把分层"框死"了,不是靠自觉。

## 面试怎么问

- "为什么 Controller 不能直接调 Repository?" → 跨层破坏职责边界,业务无法复用,单测困难。
- "三层各自负责什么?" → Controller 收发 HTTP,Service 编排业务,Repository 存取数据。
- "依赖方向能反吗?" → 不能。Spring 的 DI 机制天然支持从上往下注入,反向会让循环依赖启动报错。

## 关联

- 注入方式:[[依赖注入(DI)]] · [[构造器注入 vs 字段注入]]
- 分层注解:[[@Repository-@Service-@Controller 区别]]
- 面向接口:[[面向接口编程]]
- Web 入口:[[@RestController]]
