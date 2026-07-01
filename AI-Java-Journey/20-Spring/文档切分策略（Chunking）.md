# 文档切分策略（Chunking）

> 一句话：Chunking = RAG 检索精度的第一道闸门——把长文档拆成语义原子，让向量检索能精准命中相关段落，而不是返回整篇稀释了语义的文档。

## 它解决什么问题

**语义稀释**：一篇 50 页的《退款政策》包含 20 个不同主题（退款/换货/投诉/隐私…），用一个 1536 维向量去代表全部语义 → 向量变成模糊平均 → 搜"怎么退款"时，这个向量的相似度可能不如一个只讲退款的 800-token chunk。

同时，Embedding 模型有 **token 上限**（`text-embedding-v2` 最长 2048 tokens），超长文档直接调 `embed()` 会被截断，后半截内容根本没向量化。

**Chunking 的解法**：
```
未切分：整篇 50 页文档 → 1 个向量 → 语义稀释 + 检索不准
切分后：50 页 → TokenTextSplitter → 200 个片段 → 200 个独立向量
       → 搜索"怎么退款" → 精准命中 片段#3（退款流程）+ 片段#7（退款时限）
```

## 核心要点

### 1. chunk size 的黄金区间

| chunk size | 现象 | 结论 |
|---|---|---|
| **200 tokens** | 语义碎片化 —— "退款流程如下：第一步…"还没说完就被切断 | ❌ 太短 |
| **800 tokens** | ≈ 600 汉字，一个"完整论述"的长度 | ✅ 黄金区间 |
| **2000 tokens** | 语义重新稀释 —— 一个 chunk 塞了退款+换货+投诉三个主题 | ❌ 太长 |

**记忆**：600-1000 tokens（中文约 400-800 字）是 RAG 实验验证的最佳区间。

### 2. overlap（重叠）的作用

```
无 overlap：
  Chunk A: "退款流程如下："  ← 关键信息被切在边界
  Chunk B: "第一步，登录账户……"

有 overlap (100 tokens)：
  Chunk A: "退款流程如下：第一步，登录账户……"  ← 完整
  Chunk B: "退款流程如下：第一步，登录账户……"  ← 重复开头，但语义完整
```

**overlap 100 tokens** = 相邻 chunk 有 ~75 个汉字的重复内容，保证任何一句话都不会恰好被切在 chunk 边界。

### 3. TokenTextSplitter API（Spring AI 2.0.0）

```java
TokenTextSplitter splitter = TokenTextSplitter.builder()
    .withChunkSize(800)                    // 每块目标 token 数
    .withMinChunkSizeChars(50)             // 短于此字符数的碎片丢弃
    .withMinChunkLengthToEmbed(10)         // 短于此 token 数的碎片不灌库
    .withMaxNumChunks(10_000)              // 上限防护
    .withKeepSeparator(true)               // 保留标点符号
    .withEncodingType(EncodingType.CL100K_BASE)  // GPT-4 同款编码
    .build();

// 切分
List<Document> chunks = splitter.split(new Document(text, metadata));
```

**关键差异 vs LangChain**：

| | Spring AI 2.0.0 | LangChain Python |
|---|---|---|
| 切分单位 | Token（JTokkit） | 字符（可注入 tiktoken） |
| chunk_overlap | **无原生支持**，需手写滑动窗口 | 内置 `chunk_overlap` |
| 标点处理 | 在标点处断裂（优先语义完整） | 递归分割符列表 |

### 4. 向量化的真实时机

**关键纠正**：灌库管线里，业务代码**不显式调 EmbeddingService**：

```java
// ❌ 错误认知：先向量化再存
float[] vector = embeddingService.embed(chunkText);
// ... 存向量 ...

// ✅ 真实流程：只管塞 Document，VectorStore 内部自动向量化
vectorStore.add(List.of(document));
// ↑ VectorStore 内部调 EmbeddingModel.embed()，业务代码无感知
```

这就是 VectorStore 抽象的价值——换了 Embedding 模型，管线代码一行不动。

## Java 里怎么落地

### 项目代码（Week 5 Day 3）

**DocumentChunkingService**：封装 TokenTextSplitter + 手动实现滑动窗口重叠

```java
@Service
public class DocumentChunkingService {
    private final TokenTextSplitter tokenSplitter; // chunkSize=800, CL100K_BASE

    // 基础切分（无重叠）
    public List<Document> chunk(String text, Map<String, Object> metadata) { ... }

    // 带滑动窗口重叠（chunkSize=800, overlap=100）
    public List<Document> chunkWithOverlap(String text, Map<String, Object> metadata) { ... }
}
```

滑动窗口实现思路：
1. 按中文标点（。！？；\\n）拆句子
2. 贪心窗口：逐句加入直到 token 数接近 chunkSize
3. 下一窗口起点：从上一窗口末尾往回退，凑够 overlap tokens

**DocumentIngestionService**：编排 parse → chunk → ingest 三步

```java
@Service
public class DocumentIngestionService {
    // 文件灌库：Tika 解析 → TokenTextSplitter 切分 → VectorStore 批量入库
    public IngestionResult ingestFile(InputStream input, String filename);
    
    // 纯文本灌库：直接切分入库（跳过 Tika）
    public IngestionResult ingestText(String text, String source);
}
```

**IngestionController**：
- `POST /api/ingestion/upload` — 上传文件灌库（PDF/Word/Markdown）
- `POST /api/ingestion/text` — 纯文本灌库

### 灌库管线全景

```
文件上传 → Tika 解析（DocumentParseService, Week4）
        → TokenTextSplitter 切分（DocumentChunkingService, Day3 🆕）
        → VectorStore.add()（VectorStoreService, Day2）
           └── 内部自动调 EmbeddingModel.embed()（Day1）
        → Redis Stack 入库
```

## 踩过的坑 / 取舍

### Spring AI 2.0.0 无原生 overlap

TokenTextSplitter.Builder **没有** `withChunkOverlap()` 方法。这是 2.0.0 的设计选择（1.x 的 `TextSplitter` 有 `defaultChunkOverlap` 字段但已移除）。

**应对**：自己实现滑动窗口重叠（`chunkWithOverlap` 方法）。面试时讲这个 = "框架有局限，工程师自己补"的典型案例，很加分。

### token 估算 vs 精确计数

`chunkWithOverlap` 中 token 计算用 `字符数 / 0.75` 估算，不是调 JTokkit `Encoding.encode()`。因为对每个句子都精确编码开销太大，chunk size 本身也不需要精确到个位数（±20 tokens 不影响检索效果）。

面试追问"怎么精确计数"：调 JTokkit `encoding.encode(text).size()`。

### 大文件灌库的三个风险维度

| 维度 | 风险 | 方案 |
|------|------|------|
| 内存 | 100MB PDF → Tika 整篇解析 → OOM | `max-file-size=10MB` 硬限制 + 超大拒绝 |
| 超时 | 同步 HTTP 阻塞 | `@Async` + 任务 ID 轮询（Day 5） |
| Embedding 成本 ⭐ | 几万个 chunk × 几万次 API 调用 | `withMaxNumChunks(5000)` 上限 + 先采样评估 |

## 面试怎么问

> **"chunk size 设多少？为什么？"**

800 tokens。实验验证中文场景 600-1000 tokens 检索效果最优。太小（200）语义碎片化，太大（2000）语义重新稀释。800 tokens ≈ 600 汉字 = 一段完整论述的长度。

> **"overlap 为什么是 100？"**

防止关键信息被切在 chunk 边界。100 tokens ≈ 75 汉字，足够覆盖一个短句。面试加分点：Spring AI 2.0.0 没原生 overlap，自己写了滑动窗口。

> **"灌库管线从上传到入库经过哪几步？"**

上传 → Tika 解析为纯文本 → TokenTextSplitter 按 800 tokens 切分（带 100 overlap）→ 批量提交 VectorStore.add()（内部自动调 EmbeddingModel 向量化）→ 写入 Redis Stack。切分是检索精度的第一道闸门——切得好，后面检索事半功倍。

## 关联

- 上游：[[Embedding 向量化]]（向量从哪来）
- 上游：[[Redis Stack 向量存储]]（向量存到哪）
- 下游：[[相似度检索]]（Day 4 —— 搜出来 + 质量验证）
- 下游：[[QuestionAnswerAdvisor RAG 问答]]（Day 5 —— Advisor 自动检索 + 生成）
- 类比：[[RAG 原理与最小闭环]]（Chunking 是检索层的预处理）
- 周索引：[[Week5 索引]]
