# Week 5 索引

> **RAG v1：知识库问答最小闭环**。打通 RAG 最小闭环——把文档灌入向量库，用户自然语言提问，系统返回带原文引用来源的答案。**✅ 全部完成（Day 1-7 ✅）。**

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

## Day 4：相似检索 + 检索质量手工评估 ✅
- [[相似度检索]] —— top-K 调参、minScore 阈值、precision@K 手工评估、filterExpression 坑
- 代码：`SearchService`（检索 + 格式化 + 后置过滤）+ `SearchController`（POST /api/search + /api/search/filtered）+ `RedisSchemaInitializer`（启动自动建 FT.INDEX）
- 自测 **69 分**（Q1=15 Q2=16 Q3=14 Q4=24）
- 🔴 盲区 1：threshold 调参无法修复排序问题 —— 正确答案分数低于错误答案时，调阈值只能同时杀/放，需要混合检索+rerank（Week 6）
- 🔴 盲区 2：filterExpression 在 RedisVectorStore 中不生效 —— metadata 序列化为 JSON 字符串而非独立 hash field，需内容前缀后置过滤
- 🔴 盲区 3：metadata 检索时不返回 —— `similaritySearch` 只返回 distance/vector_score，自定义 metadata 丢失，需内嵌到内容前缀
- 🐛 踩坑：Spring AI 2.0.0 `initialize-schema=true` 在 Spring Boot 4.x 下不触发 FT.CREATE → `RedisSchemaInitializer` 手动建索引
- 关键发现：简单查询（Q1-Q3, Q5）precision 100%；复杂查询 Q4"商品坏了怎么办"因 query-doc gap 召回失败 → vocabulary gap 是 RAG 检索第一杀手
- 关键认知：检索质量 = RAG 天花板 —— 5 查询 × top-3 手工标注 → precision@3 量化报告，不凭感觉调参

## Day 5：QuestionAnswerAdvisor + RAG 完整链路 ✅
- [[QuestionAnswerAdvisor RAG 问答]] —— 手动实现 RAG 三阶段管线（检索→增强→生成），System Prompt 安全防线，空检索兜底策略
- [[RAG 原理与最小闭环]] —— Retrieve→Augment→Generate 原理、检索天花板理论、安全失败原则
- 代码：`RagService`（编排三阶段）+ `RagController`（POST /api/qa/ask + /compare + /ask-without-rag）
- 自测 **90 分**（Q1=18 Q2=22 Q3=23 Q4=27）
- 🔴 盲区 1：System Prompt vs User Prompt 的安全差异 —— 知识片段必须放 System Prompt，Transformer 注意力机制中系统指令优先级更高，防提示词注入
- 🔴 盲区 2：空检索兜底的工程决策 —— 企业知识库场景优先"安全失败"（告知不知），不降级纯 LLM（破坏可信性）
- 🔴 盲区 3：手动实现 vs 框架 Advisor 的取舍 —— Spring AI 2.0.0 已移除 QuestionAnswerAdvisor，手动实现虽多写代码但获得精确控制（Prompt 模板、异常处理、参数调优）
- 关键认知：RAG 三阶段管线的每一步都透明可控 → 出了问题知道排查方向（检索？Prompt？LLM？）

## Day 6：周末整合 —— 端到端闭环 + 测试 ✅
- 灌库管线全链路联调（upload → chunk → ingest → search → RAG ask）
- MetricsAdvisor 挂载 RagService：每次 LLM 调用产生 3 组 Micrometer 指标（duration/count/tokens）
- 代码：`RagService` 注入 `MeterRegistry` + per-request 挂载 `MetricsAdvisor`
- 集成测试：`RagServiceTest`（7 个单测——buildSystemPrompt 分支逻辑 + RagResponse）+ `RagControllerTest`（7 个测试——3 端点 HTTP 契约）
- 测试结果：85 测试总量，新增 14 个 ✅，0 失败（4 个预先存在的 ApplicationContext 错误除外）
- [[RAG 检索质量评估]] —— precision@3 综合 73.3%，Q4 "商品坏了怎么办" query-doc gap 分析

## Day 7：周总结 ✅
- [[Week5 周总结]] —— 成果盘点（11 端点 / 7 Service / 5 Controller / 85 测试）、成绩趋势（81→90→81→69→90，平均 82.2，D4→D5 反弹 +21）、15 个盲区回顾（12 已吃透 / 3 半懂）、Week 6 预告（混合检索 + rerank + Milvus）
- 全量测试验证：`mvn test` 85 总量 / 0 失败 / 4 预存错误（Redis + API Key）
- Git 状态：working tree clean，所有代码已提交
- 盲区分类：80% 已吃透，⚠️ HNSW 参数 + metadata/filterExpression 面试危险区

## 本周里程碑（目标）
- [x] Docker Redis Stack 可启动，`RedisVectorStore` 连接正常
- [x] 至少 3 篇文档成功灌库（解析→切分→向量化→入库）
- [x] `POST /api/qa/ask` 端点可调用，返回答案 + 引用来源
- [x] 答案能命中原文相关片段（非胡编）
- [x] `mvn test` 新增 RAG 相关测试全部通过（14/14）✅
- [x] MetricsAdvisor 覆盖 RAG 调用链 ✅

## 技术栈速查
- **Embedding**：百炼 `text-embedding-v2`（1536 维）→ Spring AI `EmbeddingModel`
- **向量库**：Redis Stack（Docker）→ Spring AI `RedisVectorStore` / 备选 `SimpleVectorStore`
- **切分**：Spring AI `TokenTextSplitter`（chunkSize=800, overlap=100 手写滑动窗口）
- **RAG**：`QuestionAnswerAdvisor` 自动检索 + 增强 + 生成

## 本周应沉淀的知识点
- `[[Embedding 向量化]]` · `[[Redis Stack 向量存储]]` · `[[文档切分策略（Chunking）]]` · `[[相似度检索]]` · `[[QuestionAnswerAdvisor RAG 问答]]` · `[[RAG 原理与最小闭环]]` · `[[Docker Compose 开发环境]]` · `[[RAG 检索质量评估]]` · `[[Week5 周总结]]`

## 成绩趋势
```
Day 1: 81 分 (B+)  ████████████████░░░░
Day 2: 90 分 (A-)  ██████████████████░░  ↑ +9
Day 3: 81 分 (B+)  ████████████████░░░░  ↓ -9
Day 4: 69 分 (C+)  ██████████████░░░░░░  ↓ -12 ⚠️ 答案太简略
Day 5: 90 分 (A-)  ██████████████████░░  ↑ +21 🚀 反弹，RAG 认知链路打通
Day 6: —— 分 (整合日，无自测)  ░░░░░░░░░░░░░░░░░░░░  🔧 工程整合
Day 7: —— 分 (周总结日)  ░░░░░░░░░░░░░░░░░░░░  📊 复盘+盘点

平均分: 82.2（不含 D6/D7）  评级: B+/A-
趋势: W 形波动，D4 触底后 D5 强劲反弹 +21 分
```

## 导航
- 上位：[[总图谱]]
- 上一周：[[Week4 索引]]
- 下一周：Week6 索引（待建）
- 周总结：[[Week5 周总结]]
