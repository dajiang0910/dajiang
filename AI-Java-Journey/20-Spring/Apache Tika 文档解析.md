# Apache Tika 文档解析

> Tika 是"文档解析的 JDBC"：同一行代码解析 PDF/Word/Markdown/HTML 等 1000+ 种格式。知识库接入层的第一环。

## 它解决什么问题

真实场景里知识来源是**文件**（PDF 报告、Word 需求文档、Markdown 笔记），不是纯文本。每种格式解析方式完全不同：

```
没有 Tika：
if (pdf)  → PDFBox（100 行）
else if (docx) → POI（80 行）
else if (md) → 手动去标记（30 行）

有 Tika：
tika.parseToString(inputStream, metadata);  // 一行搞定所有格式
```

Tika 内部自动做三件事：**格式检测**（读 magic bytes）→ **解析器路由**（匹配合适的解析器）→ **文本提取**。

## 核心要点

### 1. Tika 门面类

```java
Tika tika = new Tika();  // 线程安全，全局复用一个实例

// 最简用法：传 InputStream → 返回纯文本
String text = tika.parseToString(inputStream);

// 带 Metadata 的用法：传文件名帮助检测 + 获取解析后的元信息
Metadata metadata = new Metadata();
metadata.set("resourceName", "报告.pdf");
String text = tika.parseToString(inputStream, metadata);
String mimeType = metadata.get("Content-Type");  // 解析后 Metadata 被填充
```

### 2. 常见坑：detect() 和 parseToString() 不能串联

```java
// ❌ 错误：detect() 消费了 InputStream 头部字节，后续解析读不到完整内容
String type = tika.detect(inputStream);
String text = tika.parseToString(inputStream);  // ← 流已被消费！

// ✅ 正确：一次 parseToString() + Metadata 获取全部信息
Metadata metadata = new Metadata();
String text = tika.parseToString(inputStream, metadata);
String type = metadata.get("Content-Type");  // 解析后获取
```

### 3. Tika 依赖

```xml
<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
    <version>3.1.0</version>
</dependency>
<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-parsers-standard-package</artifactId>
    <version>3.1.0</version>
</dependency>
```

`tika-core` 提供 Tika 门面 + 格式检测；`tika-parsers-standard-package` 包含所有标准解析器（PDFBox、POI 等）。

### 4. 在知识库架构中的位置

```
文档上传 → Tika 解析 → 纯文本 → 结构化提取 → 向量化 → 存入向量库
           ↑                        ↑              ↑
      Week 4 Day 3           Day 1-2         Week 5
```

## Java 里怎么落地

### DocumentParseService

```java
@Service
public class DocumentParseService {
    private final Tika tika = new Tika();  // 线程安全

    public String parse(InputStream inputStream, String originalFilename) throws IOException {
        Metadata metadata = new Metadata();
        metadata.set("resourceName", originalFilename);

        String text;
        try {
            text = tika.parseToString(inputStream, metadata);
        } catch (TikaException e) {
            throw new IOException("Tika 解析失败：" + originalFilename, e);
        }

        String detectedType = metadata.get("Content-Type");
        log.info("文件类型：{}，文本长度：{}", detectedType, text.length());
        return text;
    }

    // 完整链路：解析 → AI 提取
    public NoteMetadata parseAndExtract(InputStream inputStream, String filename) throws IOException {
        String text = parse(inputStream, filename);
        return extractService.extractV2(text);  // Day 2 的 Few-shot 增强版
    }
}
```

### DocumentController

```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    @PostMapping(value = "/parse", consumes = MULTIPART_FORM_DATA_VALUE)
    public ApiResponse<DocumentParseResponse> parse(@RequestParam("file") MultipartFile file) {
        String text = parseService.parse(file.getInputStream(), file.getOriginalFilename());
        return ApiResponse.success(new DocumentParseResponse(
            file.getOriginalFilename(), "检测完成", text.length(), text));
    }

    @PostMapping(value = "/parse-and-extract", consumes = MULTIPART_FORM_DATA_VALUE)
    public ApiResponse<NoteMetadata> parseAndExtract(@RequestParam("file") MultipartFile file) {
        return ApiResponse.success(
            parseService.parseAndExtract(file.getInputStream(), file.getOriginalFilename()));
    }
}
```

### curl 验收

```bash
# 纯解析
curl -X POST http://localhost:8080/api/documents/parse \
  -F "file=@spring-ai-intro.md"

# 完整链路：解析 + AI 提取
curl -X POST http://localhost:8080/api/documents/parse-and-extract \
  -F "file=@Q3计划.txt"
```

## 面试怎么问

- **"怎么处理用户上传的各种格式文档？"** → Apache Tika 统一门面，自动检测格式 + 自动选择解析器，一行 `parseToString()` 搞定 PDF/Word/Markdown/HTML 等 1000+ 种格式。
- **"Tika 的格式检测靠什么？"** → 读文件头 magic bytes（PDF 的 `%PDF`、docx 的 `PK`（本质是 zip）），不依赖扩展名。也能通过 Metadata 传文件名辅助检测。
- **"Tika 解析大文件怎么办？"** → ① 设超时（`TikaConfig` 的 `timeoutThreshold`）② 流式解析（不全部加载到内存）③ 异步处理（上传后后台解析 + 回调通知）

## 关联

- 下游：[[BeanOutputConverter 结构化输出]] —— Tika 产出的纯文本喂给 `.entity()` 做结构化提取
- 下游：[[Prompt 工程进阶（Few-shot + Schema 约束）]] —— `parseAndExtract()` 内部调用 `extractV2()`
- 前置：[[@RestController]] —— DocumentController 遵循同样的三层架构
- 前置：[[Spring AI 起步]] —— 同一项目的 ChatClient 注入模式
- 后续：Week 5 向量化与 RAG —— 解析出的文本经 Embedding 存入向量库
