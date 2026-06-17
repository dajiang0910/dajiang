# Stream API

> 用"声明式"风格处理集合:把"怎么一步步循环"换成"我要什么结果"。

## 它解决什么问题
循环 + 累加 + 分桶这些"过程式"代码越写越长、易错、难读。Stream 让你用一串链式调用说"要什么"。

## 核心要点:三板斧
```java
list.stream()
    .filter(x -> ...)     // 过滤:把不要的踢掉
    .map(x -> ...)        // 变换:把元素换种形式
    .collect(toList());   // 收集:把流汇成结果
```
- **filter**:筛掉不满足条件的(数量减少)
- **map**:把元素转成另一种形式(数量不变,形状变)
- **collect**:把流汇成 List/Map/单个值等

## 惰性求值(面试高频,核心机制)
- `filter`、`map` 等**中间操作**只是"画图纸",**不真正执行**。
- 遇到 `collect` / `count` / `forEach` 等**终端操作**才启动流水线,元素才开始被处理。
- 验证:用 `.peek(打印)` 加到中间,删掉终端操作,程序**什么也不输出**;加上终端才输出。

## 命令式 vs 声明式
```java
// 命令式:7 行,讲"怎么做"
var m = new HashMap<String,Integer>();
for (var d : docs) { m.put(d.category(), m.getOrDefault(d.category(),0)+1); }

// 声明式:1 行,讲"要什么"
docs.stream().collect(Collectors.groupingBy(Doc::category, Collectors.counting()));
```

## Java 里怎么落地
Day3:
```java
docs.stream()
    .filter(d -> d.category().equals("财务"))
    .map(Doc::title).distinct()
    .collect(Collectors.toList());
docs.stream().collect(Collectors.groupingBy(Doc::category, Collectors.counting()));
```

## 何时用 Stream,何时用 for
- **Stream 优势**:业务逻辑复杂时,可读性高;`parallelStream()` 一行并行。
- **for 优势**:简单循环、追求极致性能、循环内 `break/continue`。
- **没有"一定更好"**,挑合适的。

## 面试怎么问
- "Stream 是惰性的吗?什么时候才执行?" → 中间操作不执行,遇终端才执行。
- "filter 和 map 区别?" → 过滤 vs 变换。

## 关联
- 配合:[[Lambda 与方法引用]]、[[Collectors]]
- 上位:[[Java 集合框架]]
- 应用:几乎所有现代 Java 代码(含 Spring AI)
