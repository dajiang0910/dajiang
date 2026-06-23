# ConcurrentHashMap

> `ConcurrentHashMap` 是线程安全的 `HashMap`。Web 服务是多线程的(Tomcat 每个请求一个线程),共享数据结构必须用并发安全的容器。

## 它解决什么问题

`HashMap` 不是线程安全的。多线程同时 `put` 会出两个经典问题:

**1) 数据丢失(所有 JDK 版本)**
线程 A 和线程 B 同时 `put`,都走到了"发现桶为空,准备写入"这一步,谁后写谁就把前一个覆盖了 → 数据静默丢失,不报错。

**2) 死循环(JDK 7 专属)**
JDK 7 的 `HashMap` 扩容用「头插法」,并发扩容时链表会形成环,下次 `get()` 进入死循环 → CPU 100%,服务假死。JDK 8 改成「尾插法」修复了这个 bug,但数据丢失问题仍在。

## ConcurrentHashMap 怎么解决

| JDK 版本 | 策略 | 粒度 |
|---|---|---|
| JDK 7 | 分段锁(Segment) | 16 段,每段一把锁 |
| JDK 8+ | CAS + synchronized(锁单个桶) | 每个桶一把锁,粒度极细 |

JDK 8+ 的 `ConcurrentHashMap`:
- **读操作几乎无锁**(`volatile` 保证可见性)
- **写操作只锁一个桶**(不锁整个 Map)
- **并发性能远超** JDK 7 的分段锁

## 代码示例

```java
// Day2 内存 Repository 用它存数据
private final Map<Long, Note> store = new ConcurrentHashMap<>();

// 多线程同时 put 也安全
store.put(1L, note1);  // 线程 A
store.put(2L, note2);  // 线程 B,不会丢数据
```

## 什么时候用哪个

| 场景 | 用什么 |
|---|---|
| 单线程(或已加锁) | `HashMap` |
| 多线程读写,需要线程安全 | `ConcurrentHashMap` |
| 多线程但只需要原子操作 | `ConcurrentHashMap` 的 `computeIfAbsent` 等原子方法 |

## 面试怎么问

- "`HashMap` 和 `ConcurrentHashMap` 区别?" → 前者非线程安全,后者线程安全;并发 `put` 前者丢数据,后者不会。
- "`ConcurrentHashMap` 怎么实现线程安全?" → JDK 8+:CAS + synchronized 锁单个桶,读几乎无锁。
- "`HashTable` 呢?" → 整张表一把锁,并发性能差,已过时,不要用。
- "JDK 7 的 `ConcurrentHashMap` 用什么?" → 分段锁(Segment),16 段各一把锁。

## 关联

- 集合框架:[[Java 集合框架]]
- 并发基础:Week 12 八股专题(JUC 并发包)
- 实战:Day2 `InMemoryNoteRepository` 用 `ConcurrentHashMap` 存数据
