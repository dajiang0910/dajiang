# Swagger 文件上传与文档测试

> `springdoc-openapi` 对 MultipartFile 处理得很自然，但有几个细节不配好 Swagger UI 会报错。Day 4 把文档上传端点配到 Swagger 可测试。

## 它解决什么问题

写好了 `POST /api/documents/parse`，在 Swagger UI 里点 "Try it out" 上传文件 → 直接拿到 JSON 结果。**不用离开浏览器就能验收整个链路**——这对前后端协作、面试演示、自测都极其高效。

## 核心要点

### 1. MultipartFile 的 Swagger 注解

springdoc-openapi 能自动识别 `@RequestParam("file") MultipartFile` 并在 Swagger UI 渲染文件选择器。但要更好的文档体验，需要手动注解：

```java
@PostMapping(value = "/parse", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
@Operation(
    summary = "文档文本解析",
    description = "上传文件，Tika 自动检测格式并提取纯文本",
    operationId = "parseDocument"  // ← 唯一的 operationId，用于代码生成
)
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "解析成功"),
    @ApiResponse(responseCode = "400", description = "文件为空"),
    @ApiResponse(responseCode = "413", description = "文件过大")
})
public ApiResponse<DocumentParseResponse> parse(
    @Parameter(description = "要解析的文档文件，最大 10 MB", required = true)
    @RequestParam("file") MultipartFile file
)
```

关键注解：
- `consumes = MULTIPART_FORM_DATA_VALUE` —— **必须**，否则 Swagger 不知道这是文件上传
- `@Parameter` —— 给 Swagger UI 的字段说明
- `@ApiResponse` —— 列出各种响应码的含义

### 2. 类名冲突坑（Day 4 踩坑）

项目 DTO 叫 `ApiResponse`，Swagger 注解也叫 `@ApiResponse`。**同一个文件里 import 两个同名类 → 编译失败**。

```java
// ❌ 不能这样 import
import com.example.notes_api.dto.ApiResponse;           // 项目 DTO
import io.swagger.v3.oas.annotations.responses.ApiResponse;  // Swagger 注解

// ✅ 做法：用完全限定名
@io.swagger.v3.oas.annotations.responses.ApiResponses({...})
@io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "200", ...)
public ApiResponse<NoteMetadata> parseAndExtract(...)
// ↑ 这个 ApiResponse 是项目 DTO
```

**教训**：给 DTO 起名时要注意不与常见注解/类冲突。如果重来一次，DTO 可以叫 `R` 或 `Result`。

### 3. Swagger UI 测试文件上传

```
1. 启动应用：mvn spring-boot:run
2. 打开：http://localhost:8080/swagger-ui.html
3. 找到 Documents → POST /api/documents/parse-and-extract
4. 点 "Try it out"
5. 在 file 字段点击 "Choose File" 选择测试文件
6. 点 "Execute"
7. 直接看到结构化 JSON 响应
```

Swagger UI 对 Multipart 的支持：
- 自动渲染文件选择器（不需要手动输路径）
- 显示响应 Schema（从 `NoteMetadata` record 自动生成）
- 可以复制 curl 命令（方便分享给同事）

### 4. Multipart 配置（application.properties）

```properties
# 单文件最大大小
spring.servlet.multipart.max-file-size=10MB
# 整个请求最大大小（含所有文件 + 字段）
spring.servlet.multipart.max-request-size=15MB
```

这两个值在 Swagger UI 看不到，但在 `MaxUploadSizeExceededException` 的 handler 里可以用 `e.getMaxUploadSize()` 拿到并返回友好提示。

### 5. @ApiResponse 的两种写法

```java
// 写法 1：只写描述（简单场景）
@ApiResponse(responseCode = "413", description = "文件过大")

// 写法 2：带示例响应体（复杂场景，前端可直接看格式）
@ApiResponse(responseCode = "413",
    description = "文件大小超限",
    content = @Content(examples = @ExampleObject(
        value = "{\"code\":413,\"message\":\"文件大小超过上限（10 MB），请压缩后重试\"}")))
```

写法 2 更完整——前端同事在 Swagger UI 里点开 413 响应就能看到错误格式，不用翻代码。

## Java 里怎么落地

```java
@RestController
@RequestMapping("/api/documents")
@Tag(name = "Documents", description = "文档解析与结构化提取")  // ← Swagger 分组名
public class DocumentController {

    @PostMapping(value = "/parse-and-extract", consumes = MULTIPART_FORM_DATA_VALUE)
    @Operation(
        summary = "文档解析 + AI 元数据提取（完整链路）",
        description = "上传文档 → Tika 解析 → AI 结构化提取 → 返回 NoteMetadata",
        operationId = "parseAndExtractDocument"
    )
    @io.swagger.v3.oas.annotations.responses.ApiResponses({
        @io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "200", description = "提取成功",
            content = @Content(schema = @Schema(implementation = NoteMetadata.class))),
        @io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "413", description = "文件大小超限（>10 MB）"),
        @io.swagger.v3.oas.annotations.responses.ApiResponse(
            responseCode = "500", description = "解析失败（文件损坏）")
    })
    public ApiResponse<NoteMetadata> parseAndExtract(
        @Parameter(description = "要解析的文档文件，最大 10 MB", required = true)
        @RequestParam("file") MultipartFile file
    ) throws IOException { ... }
}
```

## 面试怎么问

- **"你们的 API 文档怎么管理的？"** → 用 springdoc-openapi 自动生成 OpenAPI 3.0 规范文档，Swagger UI 可以直接测试所有端点包括 Multipart 文件上传。Controller 代码上的 `@Operation` `@ApiResponse` `@Parameter` 就是文档源，不与代码脱节。

- **"Swagger 里怎么测文件上传？"** → `consumes = MULTIPART_FORM_DATA_VALUE` 让 Swagger 自动识别为文件上传端点，UI 会渲染文件选择器。`@Parameter` 可以给文件字段加描述和 required 标记。

## 关联

- 上游：[[@RestController]]（控制器层基础）
- 同级：[[Apache Tika 文档解析]]（解析端点）
- 核心管线：[[结构化抽取完整链路]]（Day 4 核心产出）
- 配置：[[application.properties 配置]]
- 基础：[[Swagger 与 OpenAPI]]
