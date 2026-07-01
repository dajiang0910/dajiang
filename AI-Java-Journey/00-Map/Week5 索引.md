# Week 5 索引

> **RAG v1：知识库问答最小闭环**。本周目标：打通 RAG 最小闭环——把 Week 4 解析出的文档灌入向量库，用户用自然语言提问，系统返回带原文引用来源的答案。**🔄 进行中（Day 1 ✅，Day 2 待开始）。**

## Day 1：Embedding 直觉 + EmbeddingModel API ✅
- [[Embedding 向量化]] —— 语义坐标、余弦相似度、EmbeddingModel API
- 代码：`EmbeddingService`（embed + cosineSimilarity）+ `EmbeddingController`（/peek + /similarity）+ `EmbeddingServiceTest`（3 个测试）
- 自测 **81 分**（Q1=18 Q2=15 Q3=23 Q4=25）
- 🔴 盲区：`embed()` vs `call(EmbeddingRequest)` —— 不是"单个 vs 批量"，而是"便捷 vs 完整控制（带 EmbeddingOptions）"
- 关键认知：Embedding = 把文本变成高维空间里的坐标，语义相近 → 坐标接近 → 余弦相似度高

## Day 2：向量库起步 —— Redis Stack + VectorStore 抽象
- [[Redis Stack 向量存储]] —— Docker 部署、VectorStore 抽象、RedisVectorStore 配置
- 代码：`docker-compose.yml` + `VectorStoreConfig` + 单文档灌库验证
- 关键认知：VectorStore = 向量数据库的 JDBC，一行 `add()` 入库、一行 `similaritySearch()` 检索

## Day 3：文档切分策略 + 灌库管线
- [[文档切分策略（Chunking）]] —— TokenTextSplitter、chunk size/overlap 取舍
- 代码：`DocumentIngestionService`（解析 → 切分 → 向量化 → 批量入库）
- 关键认知：chunk 太大检索不准、太小语义不完整；overlap 防止关键信息卡在边界

## Day 4：相似检索 + 检索质量验证
- [[相似度检索]] —— similaritySearch()、top-K、minScore 阈值
- 代码：`SearchService` + 手工评估检索结果
- 关键认知：检索质量决定 RAG 上限——搜不到对的片段，LLM 再强也白搭

## Day 5：QuestionAnswerAdvisor + RAG 完整链路
- [[QuestionAnswerAdvisor RAG 问答]] —— 检索→增强→生成三阶段
- 代码：`POST /api/qa/ask` 端点（QaController + QaService + QuestionAnswerAdvisor）
- 关键认知：Advisor 在 `prompt().call()` 前自动检索 + 拼上下文，答案附引用来源

## Day 6：周末整合 —— 端到端闭环 + 测试
- 代码：灌库管线联调 + 集成测试 + MetricsAdvisor 挂 RAG 调用链
- 验收：上传文档 → 灌库 → 问答 → 带引用答案，全链路 Swagger 可演示

## Day 7：周总结
- [[Week5 周总结]] —— 成果盘点、成绩趋势、盲区回顾、Week 6 预告

## 本周里程碑（目标）
- [ ] Docker Redis Stack 可启动，`RedisVectorStore` 连接正常
- [ ] 至少 3 篇文档成功灌库（解析→切分→向量化→入库）
- [ ] `POST /api/qa/ask` 端点可调用，返回答案 + 引用来源
- [ ] 答案能命中原文相关片段（非胡编）
- [ ] `mvn test` 全绿（含新增 RAG 相关测试）
- [ ] MetricsAdvisor 覆盖 RAG 调用链

## 技术栈速查
- **Embedding**：百炼 `text-embedding-v2`（1536 维）→ Spring AI `EmbeddingModel`
- **向量库**：Redis Stack（Docker）→ Spring AI `RedisVectorStore` / 备选 `SimpleVectorStore`
- **切分**：Spring AI `TokenTextSplitter`（默认 800 tokens / overlap 100）
- **RAG**：`QuestionAnswerAdvisor` 自动检索 + 增强 + 生成

## 本周应沉淀的知识点
- `[[Embedding 向量化]]` · `[[Redis Stack 向量存储]]` · `[[文档切分策略（Chunking）]]` · `[[相似度检索]]` · `[[QuestionAnswerAdvisor RAG 问答]]` · `[[RAG 原理与最小闭环]]` · `[[Docker Compose 开发环境]]`

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week4 索引]]
- 下一周：Week6 索引（待建）
