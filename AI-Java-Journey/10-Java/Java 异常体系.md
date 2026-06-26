# Java 异常体系

> 异常按"该不该强制处理"分 checked(受检,必须 try-catch 或 throws)和 unchecked(非受检,通常由代码 bug 引起);按层级分 Error(系统级,不处理)和 Exception(程序级,要处理)。

## 它解决什么问题
区分"外部世界不可控"和"程序员自己写错",用不同策略应对:前者捕获+重试+降级,后者修代码。

## 核心要点:两维度交叉看

### 维度 1:Throwable 下的两个分支
- **Error**:系统级错误(JVM 内存溢出 OOM、栈溢出 StackOverflow),**应用程序通常不处理**,直接挂
- **Exception**:程序级错误,本文件重点
  - **checked(受检)**:`Exception` 及其子类(除 `RuntimeException`),**编译器强制** try-catch 或 throws
  - **unchecked(非受检)**:`RuntimeException` 及其子类(NPE、数组越界、算术异常),**不强制**处理,通常是代码 bug

### 维度 2:checked vs unchecked
| 维度 | checked | unchecked |
|---|---|---|
| 来源 | 外部世界不可控(IO、网络、DB) | 程序员疏忽(没判空、越界) |
| 编译器是否强制 | ✅ 强制(不处理编译报错) | ❌ 不强制 |
| 应对策略 | 捕获 + 重试 + 降级 | 修代码,别靠 try-catch 兜 |
| 典型例子 | `IOException`、`SQLException` | `NullPointerException`、`ArrayIndexOutOfBoundsException` |

## 处理方式
- **try-catch**:捕获并处理(局部兜底)
- **throws**:把异常抛给上层(不处理,声明责任)
- **try-with-resources**:自动关流(见 [[try-with-resources]])
- **自定义业务异常**:把底层异常包一层(承载上下文,统一处理)—— Day4 用了 `DataLoadException`,常搭配 [[record]] 装上下文

## 一句话记忆
> checked = 外部世界的事,不处理就编译不过;unchecked = 你代码本身写错了。

## 面试怎么问
- "checked 和 unchecked 区别?" → 编译器是否强制处理 + 来源不同
- "Error 和 Exception 区别?" → 系统级 vs 程序级,前者不处理后者要处理

## 关联
- 写法:[[try-with-resources]]
- 调试:[[SLF4J 日志]]
- 数据载体:[[record]](自定义业务异常常用 record 装上下文)
- 实战:[[HttpClient]]、Day5 实战
- Spring 实战：[[全局异常处理]] 中的 `BusinessException`（继承 RuntimeException，携带 HttpStatus）
- 测试断言：[[Mockito 单元测试]] 中的 `assertThrows(BusinessException.class, ...)`
