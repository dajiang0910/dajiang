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

## 本周里程碑（目标）
- [x] `POST /api/chat` 同步端点跑通（Day 1 代码完成）
- [x] System 角色 + PromptTemplate 端点跑通（Day 2 — 翻译/摘要/Slug 三个新端点）
- [x] `GET /api/chat/stream` 流式端点（Day 3 — SSE 流式，mvn test 13/13 全绿）
- [x] 多轮对话（消息历史）（Day 4 — ChatMemory + Advisor）
- [x] `mvn test` 全绿 13/13（Day 3-4 代码无回归）
- [ ] Swagger UI 可测试 chat 端点

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week2 索引]]
- 下一周：Week4 索引（待建）
