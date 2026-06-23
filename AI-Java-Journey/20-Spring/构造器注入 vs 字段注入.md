# 构造器注入 vs 字段注入

> **面试高频题**。结论:永远选构造器注入。字段注入是"偷偷塞",构造器注入是"明着传"。

## 代码对比

```java
// ✅ 构造器注入(本周强制)
@Service
public class NoteService {
    private final NoteRepository repo;           // final 不可变
    public NoteService(NoteRepository repo) {    // 唯一构造器,Spring 自动传参
        this.repo = repo;
    }
}

// ❌ 字段注入(不要写)
@Service
public class NoteService {
    @Autowired
    private NoteRepository repo;   // 不能 final;脱离 Spring 没法 new;依赖藏在内部
}
```

## 构造器注入的四个硬优势

| 优势 | 解释 |
|---|---|
| **final 不可变** | 注入后不会被改,天然线程安全 |
| **快速失败** | 缺依赖时**启动就报错**(Bean 创建阶段),而不是运行到一半 NPE |
| **依赖显式** | 看构造器参数就知道这个类依赖谁,藏不住 |
| **可单测** | 不用启动 Spring,直接 `new NoteService(mockRepo)` 就能测 |

## 字段注入的问题

- **不能 `final`** —— 字段注入绕过了构造器,声明 `final` 会编译报错
- **启动不报错** —— `@Autowired` 缺依赖时不是启动失败,是运行到该字段时 NPE
- **依赖隐藏** —— 只看类声明不知道它依赖什么,得翻所有字段找 `@Autowired`
- **脱离容器没法测** —— 必须用 `ReflectionTestUtils.setField()` 反射注入,麻烦且脆弱
- **鼓励 God Class** —— 字段注入太容易了,不知不觉往一个类塞 10 个依赖

## 什么时候用 Setter 注入

**几乎不用**。唯一合理场景:依赖是**可选的**(有默认行为),且你明确知道它可能不注入。实践中极少见。

## 面试怎么问

- "构造器注入 vs 字段注入?" → 构造器注入:final、快速失败、可测试、依赖显式。
- "Spring 推荐哪种?" → Spring 官方文档明确推荐构造器注入。
- "`@Autowired` 可以省略吗?" → Spring 4.3+ 单构造器时可省略 `@Autowired`,Spring 自动推断。

## 关联

- 上位概念:[[依赖注入(DI)]]
- 分层应用:[[三层架构(Controller-Service-Repository)]]
- 官方立场:Spring 官方文档 → Core Technologies → Dependency Injection → Constructor-based DI
