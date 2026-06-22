# try-with-resources

> 把"实现了 `AutoCloseable` 接口的类"放进 `try (...)`,块结束时(无论正常/异常)自动逆序调 `close()`。

## 它解决什么问题
告别手写 `finally { if (r != null) try { r.close(); } catch (...) {...} }` 这种啰嗦易错的关流代码;并且解决老式 finally 的一个隐患:close 抛异常会**覆盖**掉原始异常(真凶丢失)。

## 核心机制
```java
// AutoCloseable 接口就一个方法
public interface AutoCloseable {
    void close() throws Exception;
}
```
任何实现了它的类都能放进 `try (...)`,JVM 保证块结束时调 `close()`(成功/异常都调)。

**3 个关键认知**:
1. **能放进 `try()` 的条件**:类实现了 `AutoCloseable`。**没实现就不能放**(见下「踩坑」)。
2. **多资源逆序关闭**:`try (A a=...; B b=...)` 块结束时**先关 b 再关 a**(像栈,后开先关)。即使某个 close 抛异常,**其他资源仍保证被关闭**。
3. **不止 I/O**:`Connection/Statement/ResultSet`(JDBC)、`Lock` 封装、甚至你自己写的类,只要 `implements AutoCloseable` 都能用。

## 经典对比
```java
// ❌ 旧写法:finally 关流,啰嗦,且 close 抛异常会盖掉原始异常
BufferedReader r = null;
try {
    r = Files.newBufferedReader(path);
} catch (IOException e) { /*...*/ }
finally {
    if (r != null) try { r.close(); } catch (IOException ignored) {}
}

// ✅ 新写法:自动关,优雅
try (var r = Files.newBufferedReader(path)) {
    // ...
} catch (Exception e) { /*...*/ }
```

## Suppressed Exception(被压制异常)—— 面试深水区
如果 **try 块本身抛了异常**,然后 `close()` **又**抛异常:
- try 块的异常是**主异常**,会正常往外抛;
- `close()` 的异常被**压制(suppressed)**,挂在主异常上,靠 `mainException.getSuppressed()` 才能取到。

这正是 try-with-resources 比老式 finally 强的地方:老式 finally 里 close 抛异常会**覆盖**原始异常,真凶丢失;try-with-resources 保留主异常、压制次要异常,排查时不丢线索。

## 实战 3 例(覆盖"不止 I/O")
```java
// 1) 文件流(最常见)
try (var br = Files.newBufferedReader(path)) {
    System.out.println(br.readLine());
}

// 2) JDBC:Connection / PreparedStatement / ResultSet 都实现 AutoCloseable(Week5 用)
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* ... */ }
}   // 逆序关:rs → ps → conn

// 3) 自定义资源:任何 implements AutoCloseable 的类(Day1 亲手验证)
static class MyResource implements AutoCloseable {
    @Override public void close() { System.out.println("自动关闭"); }
}
try (MyResource a = new MyResource("A"); MyResource b = new MyResource("B")) {
    throw new RuntimeException("中途出错");
}   // 输出:关闭 B → 关闭 A(逆序),出异常也照关
```

## 踩坑
- **`HttpResponse` 不能 try-with-resources**:它**没有**实现 `AutoCloseable`,放进 `try()` 编译不过。同步 `String` 响应也不需要手动关。详见 [[踩坑-HttpResponse 不能 try-with-resources]]。
- 只有「流式 / 需要释放底层连接」的响应体(如 `InputStream` 形式)才涉及关闭。

## 面试怎么问(高频)
- "try-with-resources 原理?" → 资源类实现 `AutoCloseable`,JVM 在块结束时**逆序**调 `close()`。
- "它比 finally 关流好在哪?" → 简洁 + **Suppressed Exception**(保留主异常,不丢真凶)。
- "怎么让自己写的类支持它?" → 实现 `AutoCloseable` 接口的 `close()`。

## 关联
- 上位:[[Java 异常体系]]
- 踩坑:[[踩坑-HttpResponse 不能 try-with-resources]](反例:没实现 AutoCloseable)
- 实战:[[HttpClient]](send() 的同步响应不需要关)

---
> **修正记录**:原笔记曾把 `HttpResponse` 列为"可自动关的资源"并给了 `try (var response = client.send(...))` 示例 —— **这是错的**,`HttpResponse` 未实现 `AutoCloseable`。已删除,改记入「踩坑」。错误也是知识。
