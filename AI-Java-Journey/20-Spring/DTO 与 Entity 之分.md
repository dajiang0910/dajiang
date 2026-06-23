# DTO 与 Entity 之分

> **Entity(实体)** 代表数据库里的一条记录;**DTO(Data Transfer Object)** 代表 API 收发的数据结构。两者职责不同,不该混用。

## 核心区别

| | Entity | DTO |
|---|---|---|
| 代表什么 | 数据库里的一行 | API 请求/响应的数据结构 |
| 用什么写 | `class`(可变,ORM 要求) | `record`(不可变,天然适合传输) |
| 有没有业务逻辑 | 可以有(贫血 vs 充血模型之争) | 没有,纯数据载体 |
| 能直接返给前端吗 | 不该(会暴露内部结构/字段) | 可以,DTO 就是给前端看的 |

## 为什么要分开

假设 `Note` Entity 有个 `internalMemo` 字段(内部备注,不给用户看):
- 如果直接把 Entity 序列化成 JSON 返回 → `internalMemo` 泄露了
- 用 DTO 就能精确控制:只包含 `id`, `title`, `content`

## Java 里怎么落地

```java
// Entity:用 class(后续 ORM 要求可变 + 无参构造)
public class Note {
    private Long id;
    private String title;
    private String content;
    // getter/setter...
}

// DTO:用 record(Week1 学的,不可变 + 自动生成构造器/getter/equals/hashCode/toString)
public record NoteResponse(Long id, String title, String content) {}

// 转换:Entity → DTO
public NoteResponse toResponse(Note note) {
    return new NoteResponse(note.getId(), note.getTitle(), note.getContent());
}
```

## 什么时候用 record,什么时候用 class

| 场景 | 用什么 | 理由 |
|---|---|---|
| API 响应 DTO | `record` | 不可变、简洁、Jackson 自动序列化 |
| API 请求 DTO | `record` | 同上,Day3 加 Bean Validation 注解 |
| Entity(数据库实体) | `class` | ORM 框架(JPA/MyBatis)要求可变 + 无参构造 |
| 有复杂行为的领域对象 | `class` | 需要 setter、业务方法 |

## 面试怎么问

- "Entity 和 DTO 区别?" → Entity 代表数据库记录,DTO 代表 API 数据结构;不该混用。
- "DTO 用什么写?" → `record`(Java 16+),不可变、简洁、天然适合数据传输。
- "为什么不直接返 Entity?" → 会暴露内部字段、耦合数据库结构、无法精确控制响应内容。

## 关联

- DTO 载体:[[record]]
- 分层位置:DTO 在 Controller 层收发;Entity 在 Repository 层存取
- 实战:Day3 引入 `NoteResponse` record 做 DTO
