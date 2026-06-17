# Collectors

> Stream 终端操作 \`.collect(...)\` 的工具箱,把流汇成 List/Map/字符串/统计值。

## 它解决什么问题
\`.collect()\` 默认是"随便汇成什么"的入口,但具体怎么汇(分组?计数?拼接?转 Set?)需要 \`Collectors\` 提供策略。

## 核心要点(必须会这几个)
| 方法 | 作用 | 返回 |
|---|---|---|
| \`Collectors.toList()\` | 汇成 List | \`List<T>\` |
| \`Collectors.toSet()\` | 汇成 Set(去重) | \`Set<T>\` |
| \`Collectors.counting()\` | 数数量 | \`Long\` |
| \`Collectors.joining("、")\` | 拼成字符串 | \`String\` |
| \`Collectors.groupingBy(分类依据)\` | 分组 | \`Map<K, List<T>>\` |
| \`Collectors.groupingBy(分类, 计数)\` | 分组并统计 | \`Map<K, Long>\` |

## 经典组合:分组统计
\`\`\`java
docs.stream().collect(
    Collectors.groupingBy(Doc::category, Collectors.counting()));
// → {人事=2, 财务=3}
\`\`\`
- 第一参:按什么分类
- 第二参:同一分类下用什么方式汇总(默认是 \`toList\`,改成 \`counting()\` 就是计数)

## Java 里怎么落地
Day3 三处都用到了 Collectors:toList(财务标题)、groupingBy+counting(统计)、joining(拼知识库目录)。

## 面试怎么问
- "groupingBy 默认返回什么类型?" → \`Map<K, List<T>>\`。
- "想按分类统计数量怎么写?" → groupingBy 第二参用 counting。

## 关联
- 配合:[[Stream API]]、[[Lambda 与方法引用]]
- 应用:[[Java 集合框架]]
