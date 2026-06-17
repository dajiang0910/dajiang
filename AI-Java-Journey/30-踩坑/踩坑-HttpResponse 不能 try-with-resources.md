# 踩坑-HttpResponse 不能 try-with-resources

> JDK 21 的 `HttpResponse` 接口本身**没有** `close()` 方法,也**没** `implements AutoCloseable`,**javap -p 反编译可证**。教程里常见 `try (var response = client.send(...))` 写法在 JDK 21 上**编译都过不去**。

## 现象
```java
try (var response = client.send(request, BodyHandlers.ofString())) { ... }
// 报错:不兼容的类型: try-with-resources 不适用于变量类型
// HttpResponse<String>无法转换为AutoCloseable
```

## 根因
```bash
javap -p java.net.http.HttpResponse
# 输出里没有 close() 方法,也没有 implements AutoCloseable
```

## 怎么处理
- **同步 String 响应体**:读完 body 就不需要关,JVM 回收,**直接不写 try-with-resources**
- **流式响应**(`BodyHandlers.ofInputStream()`):关的是 **body 流本身**,不是 HttpResponse 接口
  ```java
  HttpResponse<InputStream> resp = client.send(req, BodyHandlers.ofInputStream());
  try (InputStream is = resp.body()) {  // 关的是 is
      // 读 is
  }
  ```

## 教训
1. **不要盲信教程**的 try-with-resources 写法 —— 接口在不同 JDK 上是否实现 AutoCloseable 可能有差异
2. **判能否 try-with-resources 的根本标准是"implements AutoCloseable"**,不是看源码里有没有 close 方法
3. 拿不准时 **javap -p 反编译看接口签名** —— 1 秒钟搞定,比看教程猜强 100 倍

## 关联
- 上位:[[HttpClient]]、[[try-with-resources]]
- 调试:[[SLF4J 日志]]
