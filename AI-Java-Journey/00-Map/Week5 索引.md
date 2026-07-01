# Week 5 索引

> **RAG v1：知识库问答最小闭环**。本周目标：打通 RAG 最小闭环——把 Week 4 解析出的文档灌入向量库，用户用自然语言提问，系统返回带原文引用来源的答案。**🔄 进行中（Day 1 ✅，Day 2 ✅，Day 3 ✅，Day 4 待开始）。**

## Day 1：Embedding 直觉 + EmbeddingModel API ✅
- [[Embedding 向量化]] —— 语义坐标、余弦相似度、EmbeddingModel API
- 代码：`EmbeddingService`（embed + cosineSimilarity）+ `EmbeddingController`（/peek + /similarity）+ `EmbeddingServiceTest`（3 个测试）
- 自测 **81 分**（Q1=18 Q2=15 Q3=23 Q4=25）
- 🔴 盲区：`embed()` vs `call(EmbeddingRequest)` —— 不是"单个 vs 批量"，而是"便捷 vs 完整控制（带 EmbeddingOptions）"
- 关键认知：Embedding = 把文本变成高维空间里的坐标，语义相近 → 坐标接近 → 余弦相似度高

## Day 2：向量库起步 —— Redis Stack + VectorStore 抽象 ✅
- [[Redis Stack 向量存储]] —— Docker 部署、VectorStore 抽象（= 向量数据库的 JDBC）、HNSW 索引算法、RedisVectorStore 自动配置
- 代码：`docker-compose.yml` + Maven 3 个新依赖 + `VectorStoreService`（ingest/search/delete）+ `VectorController`（POST /api/vector/ingest + /api/vector/search）+ `SimpleVectorStoreFallbackConfig`（mock 降级）
- 自测 **90 分**（Q1=18 Q2=16 Q3=19 Q4=18 Q5=19）
- 🔴 盲区：HNSW 参数（M/efConstruction/efRuntime）面试追问 + 端口冲突排查命令 `netstat -ano | findstr`
- 🐛 踩坑：Windows 原生 Redis 抢占 6379 端口 → Docker Redis Stack 连不上 → `taskkill` 解决
- 关键认知：VectorStore 接口 = 向量版 JDBC，换向量库只改 Maven 依赖 + 配置，业务代码一行不动

## Day 3：文档切分策略 + 灌库管线 ✅
- [[文档切分策略（Chunking）]] —— TokenTextSplitter、chunk size=800 / overlap=100、滑动窗口重叠手写实现
- 代码：`DocumentChunkingService`（基础切分 + 带重叠切分）+ `DocumentIngestionService`（编排 parse→chunk→ingest）+ `IngestionController`（POST /api/ingestion/upload + /api/ingestion/text）+ `DocumentChunkingServiceTest`（6 个单测）
- 自测 **81 分**（Q1=18 Q2=14 Q3=17 Q4=16 Q5=16）
- 🔴 盲区 1：chunk size 200 vs 2000 的具体危害 —— 200=语义碎片化（一句话被截断），2000=语义重新稀释（多主题混杂）
- 🔴 盲区 2：向量化发生在 VectorStore.add() 内部自动完成，业务代码不显式调 EmbeddingService —— 面试扣分点
- 🔴 盲区 3：大文件灌库风险需覆盖 4 个维度（内存 OOM / 超时 / **Embedding 成本** / 向量库容量），不只是系统资源
- 🐛 踩坑：Spring AI 2.0.0 `TokenTextSplitter` 无原生 overlap，需手写滑动窗口
- 关键认知：灌库管线 = parse（Tika）→ chunk（TokenTextSplitter）→ ingest（VectorStore.add 内部自动向量化），业务代码不显式调 Embedding

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
- [x] Docker Redis Stack 可启动，`RedisVectorStore` 连接正常
- [ ] 至少 3 篇文档成功灌库（解析→切分→向量化→入库）
- [ ] `POST /api/qa/ask` 端点可调用，返回答案 + 引用来源
- [ ] 答案能命中原文相关片段（非胡编）
- [ ] `mvn test` 全绿（含新增 RAG 相关测试）
- [ ] MetricsAdvisor 覆盖 RAG 调用链

## 技术栈速查
- **Embedding**：百炼 `text-embedding-v2`（1536 维）→ Spring AI `EmbeddingModel`
- **向量库**：Redis Stack（Docker）→ Spring AI `RedisVectorStore` / 备选 `SimpleVectorStore`
- **切分**：Spring AI `TokenTextSplitter`（chunkSize=800, overlap=100 手写滑动窗口）
- **RAG**：`QuestionAnswerAdvisor` 自动检索 + 增强 + 生成

## 本周应沉淀的知识点
- `[[Embedding 向量化]]` · `[[Redis Stack 向量存储]]` · `[[文档切分策略（Chunking）]]` · `[[相似度检索]]` · `[[QuestionAnswerAdvisor RAG 问答]]` · `[[RAG 原理与最小闭环]]` · `[[Docker Compose 开发环境]]`

## 成绩趋势
```
Day 1: 81 分 (B+)  ████████████████░░░░
Day 2: 90 分 (A-)  ██████████████████░░  ↑ +9
Day 3: 81 分 (B+)  ████████████████░░░░  ↓ -9
```

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week4 索引]]
- 下一周：Week6 索引（待建）
