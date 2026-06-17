# SLF4J 日志

> SLF4J 是日志门面,绑定实现(Logback/Log4j2)后,可以用 `Logger` 记日志;改配置文件就能切级别和输出目标,不改 Java 代码。

## 它解决什么问题
`System.out.println` 写死打到控制台,级别/目标改不了;SLF4J 可以在不改代码的情况下切换日志行为,生产环境非常关键。

## 为什么不用 System.out.println
1. **切级别**:开发期 `DEBUG`,生产期 `INFO` 以上,改 `logback.xml` 一行即可
2. **切输出目标**:打到控制台 / 文件 / 远程日志系统(Kafka/ELK),不改 Java 代码
3. **带元信息**:时间戳、线程名、类名、堆栈,排查时信息齐全
4. **性能更好**:支持按级别判断、可异步、可采样

## 5 个级别(必背)
| 级别 | 含义 | 典型场景 |
|---|---|---|
| **ERROR** | 出错影响功能 | LLM 调用失败、数据库连不上 |
| **WARN** | 可恢复的异常情况 | 重试一次成功、配额快用完 |
| **INFO** | 关键业务流程节点 | "文档入库完成"、"服务启动" |
| **DEBUG** | 详细诊断信息(开发用) | 循环里的每条数据、函数入参/返回值 |
| **TRACE** | 最详细跟踪(很少用) | 协议层字节流 |

**生产环境默认 INFO** —— 重要业务事件能看到,debug 噪声被屏蔽。

## 写法
```java
private static final Logger log = LoggerFactory.getLogger(MyClass.class);

log.info("加载 {} 篇文档", count);              // 用占位符 {},不拼字符串
log.error("加载失败:{}", e.getMessage(), e);    // 最后一个参传异常,自动打印堆栈
log.debug("响应体: {}", body);
```

## 配置文件(logback.xml)
```xml
<root level="info">                            <!-- 生产用 info -->
  <appender-ref ref="STDOUT" />
</root>
```
- 改 `<root level="debug">` 就能看到 DEBUG 输出
- 加 `<appender class="...FileAppender">` 就能同时输出到文件

## Java 里怎么落地
Day4 `log.info(...)` + 改路径触发 `log.error(...)` 带堆栈。

## 面试怎么问
- "为什么用 SLF4J 而不是 System.out?" → 切级别/切目标/带元信息,不改代码
- "日志 5 个级别什么时候用什么?" → 见上表

## 关联
- 上位:[[Java 异常体系]]
- 实战:Day4、Day5、几乎所有 Java 后端代码
