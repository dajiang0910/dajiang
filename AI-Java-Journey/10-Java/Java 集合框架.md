# Java 集合框架

> 三大核心容器:List(有序可重复)、Set(去重)、Map(键值对)。AI 应用里文档列表、检索结果、对话历史全靠它。

## 它解决什么问题
存储和操作"一组数据"。选哪个取决于:要不要有序?要不要去重?是不是键值对?

## 核心要点
| 容器 | 特点 | 典型场景 |
|------|------|----------|
| `List<T>` | 有序、可重复、按下标访问 | 文档列表、Top-K 检索结果、消息历史 |
| `Set<T>` | **去重**、无序(HashSet) | 标签集合、已处理 ID |
| `Map<K,V>` | 键值对 | 统计计数、ID→对象映射 |

- **Map 常用**:`put` / `get` / `getOrDefault(k, 默认值)` / `entrySet()` 遍历。
  ```java
  // 统计计数的惯用法
  map.put(key, map.getOrDefault(key, 0) + 1);
  ```
- **遍历**:`for (var e : map.entrySet()) { e.getKey(); e.getValue(); }`
- 创建不可变小集合:`List.of(...)` / `Set.of(...)` / `Map.of(...)`。

## Java 里怎么落地
Day2 用 `HashMap<String,Integer>` 统计各分类文档数,用 `HashSet` 去重标题。

## 面试怎么问
- "List/Set/Map 区别与场景?"
- "HashMap 怎么遍历?" → entrySet。

## 关联
- 依赖:[[泛型]](集合必须带泛型)
- 关键:[[equals 与 hashCode]](Set/Map 去重的底层依据)
- 应用:[[Optional]]
