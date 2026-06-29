# Week 4 索引

> **结构化输出 + Prompt 工程 + 文档解析**。本周目标：让 LLM 吐出 Java 可消费的结构化数据 + 打通文档解析——知识库的"接入层"雏形。Phase 2 正式启动，企业知识库智能助手开始建造。

## Day 1：BeanOutputConverter 入门 ✅
- [[BeanOutputConverter 结构化输出]] —— `.entity(Class)` 一行让 LLM 直接返回 Java Bean，3 步转换链路（Schema → 约束 → 反序列化）
- 代码：`NoteMetadata.java`（结构化输出合约）、`ExtractRequest.java`（输入 DTO）、`StructuredExtractService.java`（`.entity()` 核心调用）、`ChatController.extractMetadata()`（POST /api/extract/metadata）
- ChatControllerTest 新增 2 个测试（正常 + 空内容校验）
- 自测 **92 分**（Q1=18 Q2=18 Q3=18 Q4=20 Q5=18）
- 关键认知：`.entity()` 是主动约束（Schema 防跑偏），`.content()` + 手动解析是被动补救。`@JsonProperty(required=true)` 在 Schema 层面强制字段存在。四层防跑偏：required 约束 → System 角色 → temperature=0 → Fallback 正则兜底

## 本周里程碑（目标）
- [x] `BeanOutputConverter` 端点跑通，LLM 返回结构化 Bean（Day 1 ✅）
- [ ] Tika 能解析 PDF/Word/Markdown 三种格式
- [ ] 上传文档 → 解析 → AI 提取结构化信息完整链路
- [ ] `mvn test` 全绿（目标 32+ 测试）
- [ ] Swagger UI 可测试文档上传端点
- [ ] 本周新增 ≥4 篇 Obsidian 笔记

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week3 索引]]
- 下一周：Week5 索引（待建）
