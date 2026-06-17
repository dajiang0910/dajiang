# Lambda 与方法引用

> Lambda 是匿名函数;方法引用 \`::\` 是 Lambda 的进一步简化(语法糖)。

## 它解决什么问题
把"传一段行为"变成简单写法,让 Stream 这种"函数式风格"代码能写下去。

## 核心要点
- **Lambda 形式**:\`(参数) -> { 处理 }\`,如 \`d -> d.title()\`
- **方法引用**:\`类名::方法名\` 或 \`对象::方法名\`,如 \`Doc::title\`
  - 是 \`d -> d.title()\` 的简化 —— 当 Lambda 只是"把某个值丢给某个方法"时
  - 编译器自动把流里每个元素当参数传进去
- **底层**:Lambda 实现的是**函数式接口** —— 只有一个抽象方法的接口(如 \`Function\`、\`Predicate\`、\`Supplier\`)

## Java 里怎么落地
Day3:
\`\`\`java
.map(Doc::title)           // 等价于 .map(d -> d.title())
.filter(d -> d.category().equals("财务"))   // 复杂逻辑只能用 Lambda
\`\`\`

## 面试怎么问
- "方法引用和 Lambda 关系?" → 方法引用是 Lambda 的语法糖,前者是后者的简化写法。
- "什么场景只能写 Lambda,不能写方法引用?" → 要写多行逻辑 / 复合条件时。

## 关联
- 配合:[[Stream API]]、[[Collectors]]
