# Week 4 索引

> **结构化输出 + Prompt 工程 + 文档解析**。本周目标：让 LLM 吐出 Java 可消费的结构化数据 + 打通文档解析——知识库的"接入层"雏形。Phase 2 正式启动，企业知识库智能助手开始建造。

## Day 1：BeanOutputConverter 入门 ✅
- [[BeanOutputConverter 结构化输出]] —— `.entity(Class)` 一行让 LLM 直接返回 Java Bean，3 步转换链路（Schema → 约束 → 反序列化）
- 代码：`NoteMetadata.java`（结构化输出合约）、`ExtractRequest.java`（输入 DTO）、`StructuredExtractService.java`（`.entity()` 核心调用）、`ChatController.extractMetadata()`（POST /api/extract/metadata）
- ChatControllerTest 新增 2 个测试（正常 + 空内容校验）
- 自测 **92 分**（Q1=18 Q2=18 Q3=18 Q4=20 Q5=18）
- 关键认知：`.entity()` 是主动约束（Schema 防跑偏），`.content()` + 手动解析是被动补救。`@JsonProperty(required=true)` 在 Schema 层面强制字段存在。四层防跑偏：required 约束 → System 角色 → temperature=0 → Fallback 正则兜底

## Day 2：Prompt 工程进阶 + JSON Schema 约束 ✅
- [[Prompt 工程进阶（Few-shot + Schema 约束）]] —— Few-shot 范例 + @JsonPropertyDescription + 后校验，五层防跑偏
- 代码：`NoteMetadata.java`（+ @JsonPropertyDescription）、`StructuredExtractService.extractV2()`（Few-shot + validateAndFallback）、`ChatController.extractMetadataV2()`（POST /api/extract/metadata/v2）
- ChatControllerTest 新增 2 个测试（v2 正常提取 + 后校验修正非法 category）
- 关键认知：Few-shot 范例 > 纯文字描述（LLM 是模式补全器不是指令执行器）。@JsonPropertyDescription 写入 Schema 的 description 字段让 LLM 理解字段语义。五层防跑偏：required → description → Few-shot → System 约束 → 后校验

## Day 3：Apache Tika 文档解析入门 ✅
- [[Apache Tika 文档解析]] —— Tika 统一门面，一行代码解析 PDF/Word/Markdown 等 1000+ 格式
- 代码：`DocumentParseService.java`（Tika 封装 + 完整链路）、`DocumentParseResponse.java`（解析结果 DTO）、`DocumentController.java`（POST /api/documents/parse + POST /api/documents/parse-and-extract）
- DocumentControllerTest 新增 4 个测试（解析正常 + 空文件、提取正常 + 空文件）
- 关键认知：Tika = 文档解析的 JDBC，自动检测格式（magic bytes）+ 路由解析器。**detect() 和 parseToString() 不能串联**（detect 消费 InputStream），正确做法是一次 parseToString(stream, metadata) 并在 metadata 里拿 Content-Type。完整链路：MultipartFile → Tika → 纯文本 → extractV2() → NoteMetadata

## Day 4：结构化抽取实战 —— 端到端闭环 + Swagger 可演示 ✅
- [[结构化抽取完整链路]] —— 四段管线（MultipartFile → Tika → extractV2 → NoteMetadata），每段独立可替换，工程化五层防护
- [[Swagger 文件上传与文档测试]] —— MultipartFile 的 Swagger 注解、类名冲突坑、文件上传 UI 测试
- 代码改动：
  - 修复 `pom.xml` Tika 版本不一致（tika-core 3.2.2 + parsers 3.2.2）
  - `application.properties` 新增文件上传大小限制（`max-file-size=10MB`）
  - `GlobalExceptionHandler` 新增 `MaxUploadSizeExceededException` 处理（413 + 友好提示）
  - `DocumentController` 加文件大小校验 + 完善 @ApiResponse 注解（Swagger 文档增强）
  - **新增 `DocumentParseServiceTest`**（9 个测试，真实 Tika 解析 Markdown/HTML/XML/TXT + 边界场景）
  - 测试从 32 → **41 个全部通过**
- 关键认知：管线的每一段都是独立可替换的（Unix 管道思想）。API 设计上 parse 是原子能力，parse-and-extract 是组合能力。Swagger 的 `consumes = MULTIPART_FORM_DATA_VALUE` 让 UI 自动渲染文件选择器，不用离开浏览器就能验收完整链路。

## 本周里程碑（目标）
- [x] `BeanOutputConverter` 端点跑通，LLM 返回结构化 Bean（Day 1 ✅）
- [x] Few-shot + @JsonPropertyDescription + 后校验五层防跑偏（Day 2 ✅）
- [x] Tika 能解析 PDF/Word/Markdown 三种格式（Day 3 ✅）
- [ ] 上传文档 → 解析 → AI 提取结构化信息完整链路 ✅（Day 4 端到端闭环 + Swagger 可测试）
- [x] `mvn test` 全绿 ✅（41/41）
- [x] Swagger UI 可测试文档上传端点 ✅（Day 4 完善 @ApiResponse + MultipartFile 配置）
- [x] 本周新增 ≥4 篇 Obsidian 笔记 ✅（新增：Apache Tika、结构化抽取完整链路、Swagger 文件上传）

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week3 索引]]
- 下一周：Week5 索引（待建）
