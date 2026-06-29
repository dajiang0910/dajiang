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

## Day 2：PromptTemplate + System/User 角色（待开始）
- PromptTemplate 模板化 System/User 消息
- 待续...

## 本周里程碑（目标）
- [x] `POST /api/chat` 同步端点跑通（Day 1 代码完成）
- [ ] `GET /api/chat/stream` 流式端点
- [ ] 多轮对话（消息历史）
- [ ] `mvn test` 全绿（含 LLM 相关测试）
- [ ] Swagger UI 可测试 chat 端点

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week2 索引]]
- 下一周：Week4 索引（待建）
