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

## Day 3 实战：三种 DTO

```java
// 1. 创建请求 DTO（前端 → 后端）
public record CreateNoteRequest(
    @NotBlank(message = "标题不能为空")
    @Size(min = 1, max = 100, message = "标题长度 1-100 字符")
    String title,
    @Size(max = 10000, message = "内容最多 10000 字符")
    String content
) {}

// 2. 更新请求 DTO（和创建类似，也可独立定义）
public record UpdateNoteRequest(
    @NotBlank(message = "标题不能为空")
    @Size(min = 1, max = 100, message = "标题长度 1-100 字符")
    String title,
    @Size(max = 10000, message = "内容最多 10000 字符")
    String content
) {}

// 3. 响应 DTO（后端 → 前端）
public record NoteResponse(Long id, String title, String content, LocalDateTime createdAt) {
    // 静态工厂方法：Entity → DTO
    public static NoteResponse from(Note note) {
        return new NoteResponse(note.getId(), note.getTitle(), note.getContent(), note.getCreatedAt());
    }
}
```

**静态工厂方法 `from()` 的作用**：
- 封装 Entity → DTO 的转换逻辑
- Controller 调用 `NoteResponse.from(note)` 而非手动 new
- 防止敏感字段泄露（只暴露 DTO 定义的字段）

## 面试怎么问

- "Entity 和 DTO 区别?" → Entity 代表数据库记录,DTO 代表 API 数据结构;不该混用。
- "DTO 用什么写?" → `record`(Java 16+),不可变、简洁、天然适合数据传输。
- "为什么不直接返 Entity?" → 会暴露内部字段、耦合数据库结构、无法精确控制响应内容。
- "Entity → DTO 怎么转换?" → 静态工厂方法 `NoteResponse.from(note)`,封装转换逻辑。

## 关联

- DTO 载体:[[record]]
- 校验注解:[[Bean Validation]]（@NotBlank / @Size 放在 DTO 上）
- 分层位置:DTO 在 Controller 层收发;Entity 在 Repository 层存取
- 响应包装:[[统一响应体（ApiResponse）]] 包裹 NoteResponse 返回
- 新用途：[[Spring AI 起步]] —— ChatRequest 同样是 record DTO，不改风格直接复用
