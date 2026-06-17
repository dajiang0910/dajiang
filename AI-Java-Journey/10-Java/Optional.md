# Optional

> 用类型本身表达"这里可能没有值",逼调用者处理,避免空指针异常(NPE)。

## 它解决什么问题
Java 第一大坑是 NPE:调用 `null` 的方法直接崩。过去靠到处 `if (x != null)` 防御,易遗漏。`Optional<T>` 把"可能没有"变成类型签名的一部分。

## 核心要点
- 创建:`Optional.of(x)`(非空)、`Optional.ofNullable(x)`(可能空)、`Optional.empty()`(空)。
- 取值(推荐链式,少用 get):
  ```java
  opt.orElse("默认值")          // 有就返回值,没有返回默认
  opt.map(Doc::category).orElse("未找到")   // 先变换再取默认
  ```
- **`orElse` vs `isPresent`**:
  - `isPresent()` 返回 boolean(只判断有没有),配 `get()` 取值 —— 啰嗦。
  - `orElse(x)` 直接返回值 —— 推荐。
- 让"查不到"的方法返回 `Optional<T>` 而不是 `null`,调用方就不会忘记处理。

## Java 里怎么落地
Day2 `findByTitle` 返回 `Optional<Doc>`,查不到返回 `Optional.empty()`,调用方 `.map(...).orElse("未找到")`。

## 面试怎么问
- "Optional 解决什么问题?" → 优雅处理可能为空,防 NPE。
- "orElse 和 isPresent 区别?" → 见上。

## 关联
- 应用:[[Java 集合框架]]
- 背景:[[Java vs 旧语言]](静态类型用类型表达语义)
