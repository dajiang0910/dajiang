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

## Day 5：综合实战 —— 文档智能分析 API ✅
- [[Spring AI Advisors 拦截链]] —— Advisor 是 ChatClient 的"中间件"，责任链模式，SimpleLoggerAdvisor 在 DEBUG 日志输出完整 request/response
- [[文档智能分析综合实战]] —— Week 4 收网端点 `POST /api/documents/analyze`，四段管线：Tika 解析 → 文本统计（程序算）→ AI 元数据提取 → AI 关键句提炼（带 Advisor 观测）
- 代码改动：
  - **新增 `DocumentAnalysisResponse.java`**（综合响应 DTO：metadata + statistics + keySentences + readingTime）
  - **新增 `TextStatistics.java`**（文本统计 DTO：字符数/词数/句子数/段落数/阅读时间）
  - **新增 `DocumentAnalysisService.java`**（analyze 核心 + computeStatistics 纯计算 + extractKeySentences 带 Advisor）
  - `DocumentParseService` 新增 `ParseResult` record + `parseWithMetadata()` 方法
  - `DocumentController` 新增 `POST /api/documents/analyze`（第 18 个端点）
  - **新增 `DocumentAnalysisServiceTest`**（6 个测试，覆盖文本统计各场景）
  - `DocumentControllerTest` 新增 2 个 `/analyze` 测试
  - 测试从 41 → **49 个全部通过**
- 关键认知：
  - **成本分界线**：能程序算的绝不调 AI（computeStatistics 零 Token），该花的不省（关键句提炼需要语义理解）
  - Advisor 挂载是 **per-request** 的，通过 `.advisors(a -> a.advisors(...))` 只在当次调用生效
  - 三层可观测：SimpleLoggerAdvisor（日志）→ 自定义 MetricsAdvisor（指标）→ Micrometer Observation（链路）

## Day 6：周末整合 —— 全链路复盘 + 错题攻坚 + 自定义 Advisor 实战 ✅
- [[MetricsAdvisor 自定义指标采集]] —— 自定义 CallAdvisor，采集 LLM 调用耗时（Timer）+ Token 用量（Counter）到 Micrometer，从 SimpleLoggerAdvisor 的"调试工具"升级到"生产级可观测"
- [[ApiResponse 类名冲突（Java import 限制）]] —— Day 4 Q3 盲区沉淀为踩坑笔记：Java 没有 `import...as`，用完全限定名 or DTO 改名
- 代码改动：
  - **新增 `advisor/MetricsAdvisor.java`**（自定义 CallAdvisor，161 行）
  - **新增 `advisor/MetricsAdvisorTest.java`**（7 个测试，覆盖耗时/Token/null 防御/累加）
  - 测试从 49 → **56 个全部通过**
- 关键认知：
  - **Advisor 是 per-request 挂载，不是全局切面**——这是 Day 6 自测 Q1 最危险的误解，面试说错直接扣大分
  - **可观测三层**：SimpleLoggerAdvisor（日志/调试）→ MetricsAdvisor（指标/告警）→ Micrometer Observation（链路/分布式追踪）
  - **Timer vs Counter 选型**：耗时用 Timer（自动算 P95），Token 用量用 Counter（单调递增累计值）
  - **防御性编程**：指标采集用 try-catch 包裹，失败不影响业务调用
- 自测 **73 分**（Q1=12 Q2=13 Q3=14 Q4=16 Q5=18），Week 4 成绩触底回升

## 本周里程碑（目标）
- [x] `BeanOutputConverter` 端点跑通，LLM 返回结构化 Bean（Day 1 ✅）
- [x] Few-shot + @JsonPropertyDescription + 后校验五层防跑偏（Day 2 ✅）
- [x] Tika 能解析 PDF/Word/Markdown 三种格式（Day 3 ✅）
- [x] 上传文档 → 解析 → AI 提取结构化信息完整链路 ✅（Day 4 闭环 + Day 5 增强分析）
- [x] `mvn test` 全绿 ✅（56/56）
- [x] Swagger UI 可测试文档上传端点 ✅（Day 4 完善 @ApiResponse + MultipartFile 配置）
- [x] 本周新增 ≥4 篇 Obsidian 笔记 ✅（8 篇：Apache Tika、结构化抽取完整链路、Swagger 文件上传、Spring AI Advisors、文档智能分析综合实战、MetricsAdvisor 自定义指标采集、ApiResponse 类名冲突）
- [x] Day 6 自定义 MetricsAdvisor ✅（7 个测试，耗时 + Token 指标采集）

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week3 索引]]
- 下一周：Week5 索引（待建）

## Week 4 自测成绩趋势
```
Day 1: 92 ████████████████████░  BeanOutputConverter 入门
Day 2: 83 ████████████████░░░░  Prompt 工程进阶
Day 3: 76 ███████████████░░░░░  Apache Tika 文档解析
Day 4: 59 ███████████░░░░░░░░░  结构化抽取实战（最低谷）
Day 5: 56 ███████████░░░░░░░░░  综合实战（最低谷）
Day 6: 73 ██████████████░░░░░░  周末整合（触底回升）
```
