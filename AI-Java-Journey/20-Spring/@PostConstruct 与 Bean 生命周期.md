# @PostConstruct 与 Bean 生命周期

> `@PostConstruct` 标在一个方法上,表示"Bean 被创建并注入完所有依赖后,自动回调这个方法一次"。这是 Bean 生命周期里最常用的钩子。

## 它解决什么问题

有时 Bean 创建后需要做一次初始化:塞种子数据、建立连接池、读配置文件校验。这些事不该放在构造器里(因为依赖可能还没注入完),也不该放在业务方法里(每次调用都跑一遍)。`@PostConstruct` 就是这个"初始化回调"。

## Bean 生命周期(简化版)

```
1. 实例化        → Spring 调用构造器创建对象
2. 依赖注入      → 把所有 @Autowired / 构造器参数填好
3. @PostConstruct → 回调标记了此注解的方法 ← 你在这里做初始化
4. 就绪          → Bean 可以被使用了
5. @PreDestroy   → 容器关闭时回调(清理资源)
```

**关键:依赖注入在 `@PostConstruct` 之前完成**,所以在这个方法里可以安全使用所有注入的依赖。

## 实战:初始化种子数据

```java
@Repository
public class InMemoryNoteRepository implements NoteRepository {
    private final Map<Long, Note> store = new ConcurrentHashMap<>();
    private final AtomicLong idGen = new AtomicLong(0);

    @PostConstruct
    void seed() {   // Bean 创建 + 注入完成后自动跑一次
        save(new Note(null, "第一条笔记", "Hello Spring"));
        save(new Note(null, "第二条笔记", "构造器注入真香"));
    }
}
```

## 常见用途

| 场景 | 示例 |
|---|---|
| 种子数据 | 内存 Repository 启动时塞几条测试数据 |
| 资源初始化 | 建立连接池、加载配置文件 |
| 合法性校验 | 检查必要的配置项是否存在,缺失则启动失败 |

## 面试怎么问

- "`@PostConstruct` 什么时候调用?" → Bean 实例化 + 依赖注入之后,`@Autowired` 全填好了才回调。
- "和构造器有什么区别?" → 构造器里依赖还没注入;`@PostConstruct` 里依赖已就绪。
- "Bean 生命周期?" → 实例化 → 注入 → `@PostConstruct` → 就绪 → `@PreDestroy`。

## 关联

- Bean 注册:[[Spring IoC 容器]]
- 注入时机:[[依赖注入(DI)]]
- 实战例子:Day2 `InMemoryNoteRepository.seed()`
