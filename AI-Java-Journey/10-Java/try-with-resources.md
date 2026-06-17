# try-with-resources

> 把"实现了 `AutoCloseable` 接口的类"放进 `try (...)`,块结束时(无论正常/异常)自动调 `close()`。

## 它解决什么问题
告别手写 `finally { if (r != null) try { r.close(); } catch (...) {...} }` 这种啰嗦易错的关流代码。

## 核心机制
```java
// AutoCloseable 接口就一个方法
public interface AutoCloseable {
    void close() throws Exception;
}
```
任何实现了它的类都能放进 `try (...)`,JVM 保证块结束时调 `close()`(成功/异常都调)。

## 经典对比
```java
// ❌ 旧写法:finally 关流,啰嗦
BufferedReader r = null;
try {
    r = Files.newBufferedReader(path);
    // ...
} catch (IOException e) { /*...*/ }
finally {
    if (r != null) try { r.close(); } catch (IOException ignored) {}
}

// ✅ 新写法:自动关,优雅
try (var r = Files.newBufferedReader(path)) {
    // ...
} catch (Exception e) { /*...*/ }
```

## 常见可自动关的资源
- `BufferedReader` / `InputStream` / `OutputStream`
- `HttpResponse`(Day5 用了)
- 数据库 `Connection` / `PreparedStatement` / `ResultSet`
- `Scanner`

## Java 里怎么落地
Day5:
```java
try (var response = client.send(request, BodyHandlers.ofString())) {
    // 用 response.body()
}   // 块结束自动关 response
```

## 面试怎么问(高频)
- "try-with-resources 原理?" → 资源类实现 `AutoCloseable`,JVM 在块结束时调 `close()`。
- "怎么让自己写的类也能用 try-with-resources?" → 实现 `AutoCloseable` 接口。

## 关联
- 上位:[[Java 异常体系]]
- 实战:[[HttpClient]]、Day4-5
