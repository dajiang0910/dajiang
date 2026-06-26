# Swagger 与 OpenAPI

> 自动生成交互式 API 文档的工具。代码中的注解 → 自动生成文档页面 → 前端直接看 + 试调。

## 它解决什么问题

**没有 Swagger 时**：前端问"有哪些接口？参数是什么？"，你只能口头描述或手写文档（容易过时）。

**有 Swagger 时**：注解驱动，代码改了文档自动更新，前端直接在页面上试调接口。

## Spring Boot 接入

### 依赖（springdoc-openapi v2）

```xml
<!-- Swagger / OpenAPI 文档（springdoc v2 适配 Spring Boot 3+/4+） -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.6</version>
</dependency>
```

### 配置类

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Notes API")
                        .description("笔记管理系统 REST API")
                        .version("1.0.0"));
    }
}
```

### 访问地址

| 地址 | 作用 |
|---|---|
| `http://localhost:8080/swagger-ui.html` | 交互式文档页面（可试调） |
| `http://localhost:8080/v3/api-docs` | OpenAPI JSON（机器可读） |

## 核心注解

| 注解 | 作用 | 位置 |
|---|---|---|
| `@Tag` | Controller 分组命名 | 类上 |
| `@Operation` | 单个接口描述 | 方法上 |
| `@Parameter` | 参数描述 | 参数上 |
| `@Schema` | DTO 字段描述 | record/class 上 |

### 来源

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
```

## 实战代码（Day 6）

```java
@RestController
@RequestMapping("/api/notes")
@Tag(name = "笔记管理", description = "笔记 CRUD 操作")
public class NoteController {

    @GetMapping
    @Operation(summary = "获取所有笔记")
    public List<NoteResponse> list() { ... }

    @PostMapping
    @Operation(summary = "创建笔记")
    public ApiResponse<NoteResponse> create(
            @Valid @RequestBody CreateNoteRequest request) { ... }

    @GetMapping("/{id}")
    @Operation(summary = "根据 ID 获取笔记")
    public ApiResponse<NoteResponse> getById(@PathVariable Long id) { ... }

    @PutMapping("/{id}")
    @Operation(summary = "更新笔记")
    public NoteResponse update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateNoteRequest request) { ... }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除笔记")
    public void delete(@PathVariable Long id) { ... }
}
```

## Swagger vs OpenAPI

| | Swagger | OpenAPI |
|---|---|---|
| 关系 | 工具集（UI + Codegen） | 规范（标准格式） |
| 注解来源 | `io.swagger.v3.oas.annotations` | 同（OpenAPI 3.0 注解） |
| 历史 | Swagger 2.0 → 被 OpenAPI 收编 | OpenAPI 3.0 是当前标准 |
| 实现 | springdoc-openapi 是 Java 实现 | |

简单说：**OpenAPI 是标准，Swagger 是工具，springdoc 是 Spring 的实现**。

## 不加注解也能用

springdoc 会自动扫描 `@RestController`，即使不加 `@Tag`/`@Operation` 也能生成文档。但接口名会显示为方法名（如 `list`），不够友好。

加注解 = 文档更专业，参数说明更清晰。

## 关联
- 前置:[[@RestController]]（被文档化的对象）
- 前置:[[Bean Validation]]（校验规则也会反映到文档中）
- 官方: [springdoc-openapi](https://springdoc.org/)
- 代码: `notes-api/src/main/java/.../config/OpenApiConfig.java`
