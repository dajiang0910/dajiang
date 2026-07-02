# Week 5 周总结

> **RAG v1：知识库问答最小闭环**。Phase 2 核心周——把文档灌入向量库，用户自然语言提问，系统返回带原文引用来源的答案。从"能解析文档"升级到"能基于文档回答问题"。**6/6 里程碑全部达成 ✅。**

## 本周成果一览

| 维度 | 数量 | 明细 |
|------|------|------|
| 新增端点 | 11 | `/embedding/peek`、`/embedding/similarity`、`/vector/ingest`、`/vector/search`、`/ingestion/upload`、`/ingestion/text`、`/search`、`/search/filtered`、`/qa/ask`、`/qa/compare`、`/qa/ask-without-rag` |
| 新增 Service | 7 | `EmbeddingService`、`VectorStoreService`、`DocumentChunkingService`、`DocumentIngestionService`、`SearchService`、`RagService`、`RedisSchemaInitializer` |
| 新增 Controller | 5 | `EmbeddingController`、`VectorController`、`IngestionController`、`SearchController`、`RagController` |
| 新增 Advisor | 1 | `MetricsAdvisor`（挂载到 RagService，采集 duration/count/tokens） |
| 端点累计 | **29** | W1-2: 5, W3: 8, W4: 5, W5: +11 |
| 测试累计 | **85** | 81 通过 ✅，4 预存 ApplicationContext 错误（需 Redis + API Key） |
| 本周笔记 | **8 篇** | D1×1, D2×1, D3×1, D4×1, D5×2, D6×1, D7×1（本文） |
| 笔记累计 | **79 篇** | 12 周总进度 |

## 六天能力阶梯

```
Day 1: Embedding 向量化直觉
       EmbeddingModel API（embed + cosineSimilarity）
       1536 维语义空间 → 余弦相似度衡量"语义距离"
       关键认知：Embedding = 把文本变成高维坐标

Day 2: Redis Stack + VectorStore 抽象
       Docker 部署 Redis Stack（HNSW 索引）
       VectorStore = 向量数据库的 JDBC，换库不改代码
       踩坑：Windows 原生 Redis 抢占 6379 端口

Day 3: 文档切分 + 灌库管线
       TokenTextSplitter（chunkSize=800, overlap=100 手写滑动窗口）
       管线：parse（Tika）→ chunk（TokenTextSplitter）→ ingest（VectorStore.add 内部自动向量化）
       核心认知：业务代码不显式调 EmbeddingService

Day 4: 相似检索 + 检索质量手工评估
       top-K 调参 + minScore 阈值 + precision@K 手工标注
       5 查询 × top-3 手工标注 → precision@3 = 73.3%
       关键发现：Q4 "商品坏了怎么办" vocabulary gap → precision 0%

Day 5: RAG 三阶段管线（手动实现）
       Retrieve（SearchService）→ Augment（buildSystemPrompt）→ Generate（ChatClient）
       System Prompt 安全防线 + 空检索兜底（安全失败原则）
       三端点：/ask（RAG）、/compare（对比）、/ask-without-rag（纯 LLM）

Day 6: 周末整合 —— 端到端闭环 + 测试
       MetricsAdvisor 挂载 RagService（per-request advisor）
       RagServiceTest（7 单测）+ RagControllerTest（7 HTTP 契约测试）
       RAG 检索质量评估报告（precision@3 73.3%，Q4 gap 根因分析）
```

## 自测成绩趋势

```
90 ┤      ● D2: 90 (A-)
   │
81 ┤  ● D1: 81 (B+)   ● D3: 81 (B+)
   │
69 ┤                      ● D4: 69 (C+) ⚠️ 答案太简略
   │
   │                                  ● D5: 90 (A-) 🚀
   │                                              ● D6: —— (整合日，无自测)
   └──┬────┬────┬────┬────┬────┬────┬──
     D1   D2   D3   D4   D5   D6   D7

平均：82.2 分  评级：B+/A-  趋势：W 形波动，D4 触底后 D5 强力反弹
```

**趋势解读**：
- D1（81）开局稳健：Embedding 概念直观，API 简洁
- D2（90）本周峰值：VectorStore 抽象设计精良，上手顺畅
- D3（81）回落到开局水平：chunking 策略参数多（size/overlap），理论偏重
- D4（69）⚠️ 本周最低谷：相似检索调参 + filterExpression 坑 + precision@K 评估，**答案太简略**（自评反馈明确指出的问题）
- D5（90）🚀 最强反弹：RAG 三阶段管线把 D1-D4 全部串起来，认知链路打通
- D6 无自测（整合日：写测试 + 做评估 + 更新文档）

**D4→D5 反弹 (+21 分) 的启示**：分散学习时知识点是孤岛，整合日把它们串联成管线后，理解深度质变。这验证了 Week 4 总结中同样的规律——"分散学习 → 集中整合"策略有效。

## 本周穿越的 15 个盲区

### Day 1 盲区（1 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 1 | `embed()` vs `call(EmbeddingRequest)` | 不是"单个 vs 批量"，而是"便捷 vs 完整控制（带 EmbeddingOptions）" | ✅ 已吃透 |

### Day 2 盲区（2 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 2 | HNSW 参数（M/efConstruction/efRuntime）面试追问 | M=每个节点的最大连接数，efConstruction=构建时搜索宽度，efRuntime=查询时搜索宽度 | ⚠️ 半懂 |
| 3 | 端口冲突排查命令 `netstat -ano \| findstr` | Windows 原生 Redis 抢占 6379 → Docker Redis Stack 连不上 → `taskkill` 解决 | ✅ 已吃透 + 踩坑笔记 |

### Day 3 盲区（3 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 4 | chunk size 200 vs 2000 的具体危害 | 200=语义碎片化（一句话被截断），2000=语义重新稀释（多主题混杂） | ✅ 已吃透 |
| 5 | 向量化发生在 VectorStore.add() 内部自动完成 | 业务代码不显式调 EmbeddingService —— 面试扣分点 | ✅ 已吃透 |
| 6 | 大文件灌库风险需覆盖 4 个维度 | 内存 OOM / 超时 / **Embedding 成本** / 向量库容量，不只是系统资源 | ⚠️ 半懂 |
| 7 | Spring AI 2.0.0 TokenTextSplitter 无原生 overlap | 需手写滑动窗口 | ✅ 已吃透 + 踩坑笔记 |

### Day 4 盲区（3 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 8 | threshold 调参无法修复排序问题 | 正确答案分数低于错误答案时，调阈值只能同时杀/放，需要混合检索+rerank（Week 6） | ✅ 已吃透 |
| 9 | filterExpression 在 RedisVectorStore 中不生效 | metadata 序列化为 JSON 字符串而非独立 hash field，需内容前缀后置过滤 | ⚠️ 半懂 |
| 10 | metadata 检索时不返回 | similaritySearch 只返回 distance/vector_score，自定义 metadata 丢失，需内嵌到内容前缀 | ⚠️ 半懂 |
| 11 | initialize-schema=true 在 Spring Boot 4.x 下不触发 FT.CREATE | RedisSchemaInitializer 手动建索引 | ✅ 已吃透 + 踩坑笔记 |

### Day 5 盲区（3 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 12 | System Prompt vs User Prompt 的安全差异 | 知识片段必须放 System Prompt，Transformer 注意力机制中系统指令优先级更高，防提示词注入 | ✅ 已吃透 |
| 13 | 空检索兜底的工程决策 | 企业知识库场景优先"安全失败"（告知不知），不降级纯 LLM（破坏可信性） | ✅ 已吃透 |
| 14 | 手动实现 vs 框架 Advisor 的取舍 | Spring AI 2.0.0 已移除 QuestionAnswerAdvisor，手动实现虽多写代码但获得精确控制 | ✅ 已吃透 |

### Day 6 踩坑（1 个）

| # | 盲区 | 根因 | 状态 |
|---|------|------|------|
| 15 | Jackson record 反序列化 primitive 字段缺失报错 | Record 的 canonical constructor 要求所有字段必须存在，primitive 不能 default 为 null → HttpMessageNotReadableException | ✅ 已吃透 + RagControllerTest 修复 |

### 盲区分类汇总

```
已吃透：12/15（80%） ████████████████████░░░░
半懂/需强化：3/15（20%）—— #2 HNSW参数、#6 大文件4维度、#9-10 metadata/filterExpression
```

**面试危险区**（面试官追问时容易露馅的）：
- ⚠️ #2 HNSW 参数：面试官问"你用的向量库索引算法是什么？参数怎么调的？"——必须能说出 M/efConstruction/efRuntime 的含义和调参方向
- ⚠️ #9-10 metadata/filterExpression：面试官问"你怎么按来源过滤检索结果？"——必须能解释为什么 RedisVectorStore 的 filterExpression 不生效 + 你是怎么绕过的

## 本周核心认知（面试可讲）

### 1. RAG 三阶段管线的本质
> "Retrieve → Augment → Generate。每一阶段独立可替换：检索可以从纯向量升级到混合检索（Week 6），Prompt 模板可以调优，LLM 可以从 qwen-plus 换成更强的模型。关键是每一阶段出问题都能量化定位——检索 precision 低就优化检索，Prompt 质量差就调 Prompt，LLM 幻觉就换模型或加约束。"

### 2. 检索质量 = RAG 天花板
> "RAG 答案质量 ≤ 检索质量。Q4 '商品坏了怎么办' precision 0% 验证了这个公理：检索没捞出相关文档 → Prompt 中无相关知识 → LLM 再强也白搭。这就是为什么 Week 6 要上混合检索 + rerank——先抬高天花板。"

### 3. VectorStore 抽象 = 向量数据库的 JDBC
> "Spring AI 的 VectorStore 接口相当于数据库的 JDBC——换向量库只改 Maven 依赖 + 配置，业务代码一行不动。Redis Stack 起步、PGVector 备选、Milvus 进阶。面试时讲清楚'为什么先选 Redis Stack（已有 Redis 基础设施、HNSW 够用）以及什么时候该换 Milvus（百万级以上、需要 GPU 索引、需要多模态检索）'。"

### 4. 安全失败 vs 降级兜底
> "空检索时我选择了安全失败——告知用户'暂无相关信息'——而不是降级为纯 LLM 编造。企业知识库场景中，'不知道'比'编一个'更可信。纯 LLM 降级会破坏用户对知识库的信任——上次能查到具体条款，这次居然给了通用建议？这对 toB 产品是致命的。"

### 5. per-request Advisor 挂载的设计意图
> "MetricsAdvisor 通过 `.advisors(a -> a.advisors(metricsAdvisor))` 每次请求新建并挂载——不是全局注册。因为 advisor 内部持有计时的 ThreadLocal 状态，全局复用在并发场景下会互相污染。这也是 Spring AI CallAdvisor 链的设计哲学：advisor 是请求级别的中间件，不是应用程序级别的切面。"

### 6. 灌库管线中的隐式向量化
> "很多人的误区是灌库时要显式调 EmbeddingService.embed()，实际上 VectorStore.add(Document) 内部自动调用了 EmbeddingModel。这就像 JPA 的 `repository.save()`——你不需要手写 INSERT SQL。但面试时如果被问'向量化发生在哪一步'，答不上来就是扣分项。"

## 最终项目进度

```
Phase 1 (W1-3): Java/Spring 地基 + 第一次接 LLM ✅
Phase 2 (W4-6): 结构化输出 + RAG
  ├── Week 4: 文档解析 + 结构化抽取 ✅
  ├── Week 5: RAG v1 知识库问答 ✅  ← 你在这里
  └── Week 6: RAG v2 检索质量工程化 ⬜
Phase 3 (W7-9): Agent + 工作流 + MCP
Phase 4 (W10-12): 工程化 + 求职冲刺
```

**当前进度**：Phase 2 进度 2/3。下周进入 Week 6（RAG v2）——混合检索 + rerank + Query 改写 + Milvus，把 precision@3 从 73.3% 提到 ≥85%。

## Week 6 预告

| 维度 | 内容 |
|------|------|
| **核心主题** | RAG 检索质量工程化 —— 混合检索（BM25 + 向量）+ rerank + Query 改写 |
| **新向量库** | Milvus（从 Redis Stack 升级到专业向量数据库） |
| **关键指标** | precision@K ≥ 85%、评测集扩展至 20+ 查询 |
| **待解决问题** | Q4 vocabulary gap（"商品坏了" vs "质量问题"）、filterExpression metadata 过滤 |
| **技术栈新增** | BM25（Redis FT.SEARCH 或独立实现）、Rerank 模型、Milvus Java SDK、Query 改写 LLM 调用 |

## 关联

- 上位：[[Week5 索引]]
- 总图：[[总图谱]]
- 上一周：[[Week4 周总结]]
- 下一周：Week6 索引（待建）
- 核心笔记链：
  - [[Embedding 向量化]] → [[Redis Stack 向量存储]] → [[文档切分策略（Chunking）]] → [[相似度检索]] → [[RAG 原理与最小闭环]] → [[QuestionAnswerAdvisor RAG 问答]] → [[RAG 检索质量评估]]
  - 可观测链：[[MetricsAdvisor 自定义指标采集]]
- 踩坑笔记：
  - [[踩坑-Windows 原生 Redis 抢占 6379 端口]]（Day 2）
  - [[踩坑-TokenTextSplitter 无原生 overlap]]（Day 3）
  - [[踩坑-RedisVectorStore initialize-schema 不触发]]（Day 4）
  - [[踩坑-Jackson record 反序列化 primitive 字段缺失]]（Day 6）
