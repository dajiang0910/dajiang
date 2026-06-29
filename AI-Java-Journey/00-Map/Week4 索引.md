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

## 本周里程碑（目标）
- [x] `BeanOutputConverter` 端点跑通，LLM 返回结构化 Bean（Day 1 ✅）
- [x] Few-shot + @JsonPropertyDescription + 后校验五层防跑偏（Day 2 ✅）
- [x] Tika 能解析 PDF/Word/Markdown 三种格式（Day 3 ✅）
- [ ] 上传文档 → 解析 → AI 提取结构化信息完整链路
- [x] `mvn test` 全绿（28/28）✅
- [ ] Swagger UI 可测试文档上传端点
- [ ] 本周新增 ≥4 篇 Obsidian 笔记（2/4）

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week3 索引]]
- 下一周：Week5 索引（待建）
