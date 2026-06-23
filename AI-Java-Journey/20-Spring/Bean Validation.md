# Bean Validation（参数校验）

> **Bean Validation** 是 Java 标准的参数校验框架，通过注解声明校验规则，配合 `@Valid` 自动触发。Spring Boot 3.x 需要单独引入 `spring-boot-starter-validation`。

## 为什么需要校验

没有 Bean Validation 时，手动校验：
```java
@PostMapping
public NoteResponse create(@RequestBody CreateNoteRequest request) {
    if (request.title() == null || request.title().isBlank()) {
        throw new RuntimeException("标题不能为空");
    }
    if (request.title().length() > 100) {
        throw new RuntimeException("标题太长");
    }
    // ...每个接口都要重复写
}
```

有了 Bean Validation，注解声明规则 + `@Valid` 自动触发：
```java
@PostMapping
public NoteResponse create(@Valid @RequestBody CreateNoteRequest request) {
    // 校验通过才会进来，失败自动返回 400
}
```

## 依赖配置

Spring Boot 3.x 需要**单独引入**（2.x 内置在 web starter 里）：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

## 常用校验注解

| 注解 | 作用 | 示例 |
|---|---|---|
| `@NotBlank` | 字符串不能为 null/空/空白 | `@NotBlank(message = "标题不能为空")` |
| `@NotEmpty` | 集合/字符串不能为 null 或空 | `@NotEmpty List<String> tags` |
| `@NotNull` | 不能为 null | `@NotNull Long categoryId` |
| `@Size` | 长度/大小限制 | `@Size(min = 1, max = 100)` |
| `@Min` / `@Max` | 数值范围 | `@Min(0) @Max(100) int score` |
| `@Email` | 邮箱格式 | `@Email String email` |
| `@Pattern` | 正则匹配 | `@Pattern(regexp = "^1\\d{10}$") String phone` |

## 在 DTO 上使用（record）

```java
public record CreateNoteRequest(
    @NotBlank(message = "标题不能为空")
    @Size(min = 1, max = 100, message = "标题长度 1-100 字符")
    String title,

    @Size(max = 10000, message = "内容最多 10000 字符")
    String content
) {}
```

## 在 Controller 上触发

```java
// @Valid 触发校验，@RequestBody 接收 JSON
@PostMapping
public ApiResponse<NoteResponse> create(@Valid @RequestBody CreateNoteRequest request) {
    // 校验通过才执行
}
```

校验失败 → 抛 `MethodArgumentNotValidException` → 被 [[全局异常处理]] 捕获 → 返回 400 + 错误信息。

## @Valid vs @Validated

| | `@Valid` | `@Validated` |
|---|---|---|
| 来源 | Jakarta Bean Validation | Spring 扩展 |
| 用途 | 触发校验 + 支持分组 | 触发校验 + 支持分组 + 可用在类上 |
| 场景 | Controller 方法参数（推荐） | 需要分组校验时 |

Day 3 用 `@Valid` 就够了。

## 面试怎么问

- "Spring 怎么做参数校验？" → Bean Validation 注解声明规则 + `@Valid` 自动触发
- "校验失败怎么处理？" → 抛 `MethodArgumentNotValidException`，全局异常处理器捕获返回 400
- "Spring Boot 3.x 需要额外配置吗？" → 需要引入 `spring-boot-starter-validation`（2.x 内置）

## 关联

- 搭配使用：[[统一响应体（ApiResponse）]]、[[全局异常处理]]
- 注解放在：[[DTO 与 Entity 之分]] 的 record 上
- 触发位置：[[@RestController]] 的方法参数
