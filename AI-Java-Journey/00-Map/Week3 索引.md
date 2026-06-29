# Week 3 索引

> **第一次接 LLM — Spring AI**。本周目标：把 LLM 变成后端的"普通依赖"，产出 `POST /api/chat`（同步）+ `GET /api/chat/stream`（流式）+ 多轮对话。工程在 `notes-api/` 中增量开发。

## Day 1：Spring AI 起步 + ChatClient Hello World ✅
- [[Spring AI 起步]] —— LLM 只是后端的普通依赖，ChatClient 方法链，阿里百炼配置
- 代码：`ChatService.java`、`ChatController.java`、`ChatRequest.java`
- pom.xml：`spring-ai-starter-model-openai:2.0.0`
- 自测加权 **73 分**（Q1=18 Q2=15 Q3=20 Q4=20 Q5=0）
- Q2 纠偏：`.call()` 返回 `CallResponseSpec`（不是 `ChatClientResponse`）
- Q5 纠偏：Spring AI 2.0 模块拆分 — 模型实现 / 自动配置 / ChatClient 三模块独立 + starter 聚合
- 关键认知：阿里百炼能用 OpenAI 模块是因为 `base-url` 指向百炼兼容端点

## Day 2：PromptTemplate + System/User 角色 ✅
- [[System 角色与消息类型]] —— system/user/assistant 三种角色，`.system()` 约束 AI 行为
- [[PromptTemplate 模板化提示词]] —— 命名占位符 `{name}` 替代字符串拼接，三步操作（创建→填充→渲染）
- 代码：`TranslateRequest.java`、`SummarizeRequest.java`、`SlugRequest.java`（3 个新 DTO）
- ChatService 新增：`translate()`（system 角色）、`summarize()`（PromptTemplate）、`generateSlug()`（两者结合）
- ChatController 新增：`POST /api/chat/translate`、`POST /api/chat/summarize`、`POST /api/chat/slug`
- 自测 **100 分**（Q1=20 Q2=20 Q3=20 Q4=20 Q5=20） 🎉
- 关键认知：system 管"怎么答"（角色/格式），PromptTemplate 管"答什么"（参数化输入），两者互补不冲突

## Day 3：SSE 流式对话（stream() + Flux） ✅
- [[SSE 流式对话]] —— `.stream()` 替代 `.call()`，`Flux<String>` 异步非阻塞，`text/event-stream`
- 代码：`ChatService.streamChat()`（.stream().content()）、`ChatController.streamChat()`（GET + produces）
- 自测 **80 分**（Q1=20 Q2=0 Q3=20 Q4=20 Q5=20）
- Q2 纠偏：SSE Content-Type 是 `text/event-stream`，不是 `application/octet-stream`（后者是文件下载二进制流）
- 关键认知：同一个 ChatClient，`.call()→String→JSON` vs `.stream()→Flux<String>→SSE`，换一个调用方法改变整个通信模式

## Day 4：多轮对话（消息历史 + ChatMemory + Advisor） ✅
- [[多轮对话（消息历史）]] —— LLM 无状态 → ChatMemory 存历史 → MessageChatMemoryAdvisor 自动注入/提取
- 代码：`ChatMultiTurnRequest.java`（conversationId + message）、`ChatService.chatMultiTurn()`（Advisor 配置）、`ChatController.chatMultiTurn()`（POST /api/chat/multi-turn）
- 自测 **100 分**（Q1=20 Q2=20 Q3=20 Q4=20 Q5=20）🎉
- 关键认知：Advisor 模式 = LLM 调用的 Filter/Interceptor，请求前注入历史 → 响应后保存本轮。conversationId 区分会话，同一 ID 共享记忆、不同 ID 互相隔离
- ChatMemory 由 `ChatMemoryAutoConfiguration` 自动配置（classpath 有 spring-ai-autoconfigure-model-chat-memory 即创建 InMemory Bean），零配置直接注入

## Day 5：超时重试 + Token 成本 ✅
- [[超时重试与Token成本]] —— 三层防护：超时掐断（防线程耗尽）+ 指数退避重试（防瞬时故障）+ Token 监控（防成本异常）
- 代码：`application.properties` 新增三块配置（超时/重试/观测）、`ChatCostResponse.java`（reply + TokenUsage）、`ChatService.chatWithCost()`（用 `.chatClientResponse()` 替代 `.content()` 获取 Usage）、`ChatController.chatWithCost()`（POST /api/chat/with-cost）
- 自测 **100 分**（Q1=20 Q2=20 Q3=20 Q4=20 Q5=20）🎉
- 关键认知：`.content()` 是快捷方式只拿文本，`.chatClientResponse()` 是完整路径能拿到 Token 用量 + 限流信息 + 模型名称。重试自动配置 `SpringAiRetryAutoConfiguration` 零代码生效。4xx 不重试（请求本身有问题），5xx/网络异常重试（瞬时故障）

## Day 6：综合实战（智能笔记发布助手） ✅
- [[综合实战（智能笔记助手）]] —— 多步 AI 调用链 + Token 成本聚合，Day 1-5 全部能力整合
- 代码：`SmartNoteRequest.java`（content）、`SmartNoteResponse.java`（title + summary + translation + TokenUsage）、`ChatService.smartNote()`（3 步调用 + Token 累加）、`ChatController.smartNote()`（POST /api/chat/smart-note）、`ChatControllerTest.java`（12 个新测试覆盖全部 8 个 chat 端点）
- 自测 **80 分**（Q1=20 Q2=20 Q3=20 Q4=20 Q5=0）
- Q5 纠偏：面试时最弱的回答是「我调过 API，能返回结果」—— 缺少设计思考、工程化、量化。强回答要包含具体项目 + 设计决策 + 技术难点 + 解决方案
- 关键认知：多步调用链每步用 `.chatClientResponse()` 获取 Usage，三步累加得链路总成本。三步互不依赖可并行化（CompletableFuture），串行版更易调试。`@WebMvcTest` + `@MockitoBean` 不加载真实 ChatClient，无需 API Key

## Day 7：周总结 + 复盘 ✅
- 本周 6 天全景回顾：8 个 chat 端点 + 24/24 测试全绿 + 61 篇笔记
- 模拟面试 6 题：ChatClient 集成方式 / `.content()` vs `.chatClientResponse()` / SSE 底层变化 / 多轮记忆原理 / 三层防护 / 串行 vs 并行权衡
- 综合自评 **87 分**，薄弱点：SSE/WebFlux 底层原理、面试 STAR 话术包装、并行编排实战
- Week 3 目标全部达成：LLM 变成后端的"普通依赖"，所有里程碑 ✅
- 面试准备：本周 6 天累计覆盖 10+ 道面试题，从 API 使用 → 设计决策 → 工程化权衡形成完整回答链

## 本周里程碑（目标）
- [x] `POST /api/chat` 同步端点跑通（Day 1 代码完成）
- [x] System 角色 + PromptTemplate 端点跑通（Day 2 — 翻译/摘要/Slug 三个新端点）
- [x] `GET /api/chat/stream` 流式端点（Day 3 — SSE 流式，mvn test 13/13 全绿）
- [x] 多轮对话（消息历史）（Day 4 — ChatMemory + Advisor）
- [x] 超时重试 + Token 成本（Day 5 — 三层防护）
- [x] `mvn test` 全绿 24/24（Day 6 新增 12 个 ChatController 测试）
- [x] Swagger UI 可测试 chat 端点（Day 6 完成 ✅ — 8 个端点全部可测试）

## Week 3 成绩单

| Day | 主题 | 自测 | 核心产出 |
|---|---|---|---|
| D1 | Spring AI 起步 | 73 | ChatService + ChatController 基础骨架 |
| D2 | System 角色 + PromptTemplate | 100 | 翻译/摘要/Slug 三个端点 |
| D3 | SSE 流式对话 | 80 | stream() + Flux\<String\> + text/event-stream |
| D4 | 多轮对话 | 100 | ChatMemory + Advisor，conversationId 会话隔离 |
| D5 | 超时重试 + Token 成本 | 100 | 三层防护，chatClientResponse() 获取 Usage |
| D6 | 综合实战 | 80 | 3 步 AI 调用链 + Token 聚合 + 12 个控制器测试 |
| D7 | 周总结复盘 | — | 6 道模拟面试 + 综合自评 87 分 |
| **平均** | | **88.8** | **8 端点 + 24 测试 + 7 个 ChatService 方法** |

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week2 索引]]
- 下一周：Week4 索引（待建）
