# record

> 一行声明一个**不可变数据类**,编译器自动生成构造器、访问器、`equals`、`hashCode`、`toString`。

## 它解决什么问题
替代过去写一个数据类要手写一堆样板代码(字段 + 构造器 + getter + equals + hashCode + toString)的繁琐。

## 核心要点
- 语法:`record Person(String name, int age) {}`
- 自动生成:
  - **全参构造器**(canonical constructor)
  - **访问器** `name()` / `age()` —— ⚠️ 注意是 `name()`,**不是** `getName()`(和传统 JavaBean 的区别)
  - `equals()` / `hashCode()`(**按所有字段值**比,天然满足 [[equals 与 hashCode]] 铁律)
  - `toString()`

## 为什么"自动 equals/hashCode"是杀手锏
传统写数据类最容易踩的坑就是**只重写 equals 不重写 hashCode** → Set/Map 去重失败。
record 编译器按字段生成 equals+hashCode,**保证"equals 相等 ⇒ hashCode 一定相等"**,直接把这条铁律变成"免费"。所以放自定义数据去重时,record 几乎不会犯错。

- 字段是 **final、不可变**:适合表达"值对象 / DTO / 一次性数据载体"。

## Java 里怎么落地
Day1 代码:
```java
record Person(String name, int age) {}
// 用:
var p = new Person("Ada", 30);
p.name();  // "Ada"  —— 访问器没有 get 前缀
```
后面做企业知识库助手时,LLM 结构化输出常映射到 record。

## 面试怎么问
- "record 帮你生成了什么?" → 构造器、访问器、equals、hashCode、toString。
- "record 和普通类区别?" → 不可变、字段 final、访问器无 get 前缀、适合数据载体。

## 关联
- 对照:[[var 与静态类型]]
- 上位:[[Java vs 旧语言]]
- 铁律:[[equals 与 hashCode]](自动按字段生成,天然满足去重铁律)
