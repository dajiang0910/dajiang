# RAG 原理与最小闭环

> 一句话：RAG = 先查资料再回答，让 LLM 的回答有据可查，而不是凭空编造。

## 它解决什么问题

LLM 有两大缺陷：**知识截止日期**（训练数据有截止时间）和**幻觉**（对不知道的事瞎编）。RAG（Retrieval-Augmented Generation，检索增强生成）在 LLM 回答之前，先从外部知识库检索相关信息，把检索结果和问题一起给 LLM，让它的回答基于"查到的资料"而非"训练记忆"。

**不用 RAG 会怎样？** 问 LLM "公司今年的退款政策是什么"，它要么说"我的知识截止到…"（无法回答），要么编一套看似合理但完全错误的政策（幻觉）。

## 核心要点

### 1. 三阶段管线：Retrieve → Augment → Generate

```
用户提问 "如何退款？"
       │
  ┌────▼────────────────────────────────────┐
  │ 1. Retrieve（检索）                       │
  │    SearchService → VectorStore           │
  │    "如何退款" → 向量化 → HNSW KNN 搜索    │
  │    返回 top-K 相关文档片段                 │
  └────┬────────────────────────────────────┘
       │ List<Document>（含内容 + 来源 + 相似度）
  ┌────▼────────────────────────────────────┐
  │ 2. Augment（增强）                        │
  │    把检索结果注入 System Prompt            │
  │    "基于以下知识片段回答：[片段1][片段2]"    │
  │    + 行为约束："只基于知识片段，不编造"     │
  └────┬────────────────────────────────────┘
       │ 增强后的 Prompt（System + User）
  ┌────▼────────────────────────────────────┐
  │ 3. Generate（生成）                       │
  │    ChatClient.prompt().call()            │
  │    LLM 基于注入的上下文生成带引用的答案     │
  └────┬────────────────────────────────────┘
       │ String answer（含"根据片段1…"等引用）
       ▼
  返回给用户
```

### 2. 为什么知识片段放 System Prompt 而不是 User Prompt

| 位置 | 安全性 | 风险 |
|------|--------|------|
| System Prompt | ✅ 高 | 模型优先遵守系统指令 |
| User Prompt | ❌ 低 | 用户可注入"忽略之前的指令"来覆盖 |

**面试说法**：System Prompt 在 Transformer 的注意力机制中优先级更高，模型更倾向于遵守系统指令。把知识片段放在 System Prompt 中有效防御提示词注入——用户无法通过输入"忘记上面的资料"来绕过知识库约束。

### 3. RAG vs 纯 LLM（无 RAG）

| | 纯 LLM | RAG |
|---|---|---|
| 知识来源 | 训练数据（有截止日期） | 知识库（实时更新） |
| 准确性 | 可能幻觉 | 有据可查 |
| 引用来源 | ❌ 无 | ✅ 可标注片段编号 |
| 时效性 | 训练时的数据 | 灌库即生效 |
| 私有知识 | ❌ 不知道 | ✅ 可灌入企业文档 |
| 延迟 | 快（仅 LLM 推理） | 慢（检索 + LLM 推理） |
| 成本 | 低（仅 LLM Token） | 高（Embedding + 检索 + LLM Token） |

### 4. 检索质量 = RAG 天花板

这是 Week 5 最核心的认知：

```
RAG 答案质量 ≤ 检索质量

如果检索没把相关信息捞出来 → Prompt 里没有正确答案 →
LLM 再强也白搭（巧妇难为无米之炊）
```

**这解释了一个关键现象**：同样是 RAG，"如何退款"答得很好，"商品坏了怎么办"答不出来——不是 LLM 的问题，是检索层没召回相关信息（query-doc gap）。

### 5. 检索失败时的降级策略

```
检索命中 → 严格基于知识片段回答
检索空 → 明确告知"知识库暂无相关信息" + 给出排查建议
       → 是否降级到纯 LLM 回答？（工程决策：保守场景不要）
```

## Java 里怎么落地

```java
@Service
public class RagService {
    // 三阶段管线
    public RagResponse ask(String question) {
        // 1. Retrieve
        SearchResult ctx = searchService.searchAndFormat(question, topK, threshold);
        
        // 2. Augment
        String prompt = buildSystemPrompt(ctx);  // 注入知识 + 行为约束
        
        // 3. Generate
        String answer = chatClient.prompt().system(prompt).user(question).call().content();
        
        return new RagResponse(question, answer, ctx.hits());
    }
}
```

**关键设计**：
- `buildSystemPrompt` 区分"有结果"和"无结果"两种分支，避免空字符串拼入 Prompt 导致 LLM 自由编造
- System Prompt 放角色 + 知识 + 约束，User Prompt 只放用户原始问题
- 答案附 `sources` 数组（含 source、chunkIndex、score），可追溯

## 面试怎么问

> **"讲讲你对 RAG 的理解"**

用三阶段管线回答：用户问 → 检索（VectorStore KNN）→ 增强（知识注入 System Prompt）→ 生成（LLM 基于注入知识回答）。穿插三个关键设计决策：为什么放 System Prompt（防注入）、为什么区分有无检索结果（防幻觉）、为什么检索质量决定 RAG 上限（Day 4 实验数据）。

> **"RAG 有什么局限性？"**

1. 检索质量瓶颈（query-doc gap 导致召回失败）→ Week 6 混合检索 + rerank
2. 多跳推理困难（需要综合多个片段回答的问题）→ Week 8 Agent 多步工作流
3. 长上下文窗口压力（片段太多 token 不够）→ top-K 调参 + 片段压缩
4. 检索延迟增加用户等待时间 → 缓存热点查询

## 关联

- 上游：[[Embedding 向量化]]（查询向量化 → 语义检索）
- 上游：[[Redis Stack 向量存储]]（检索的物理载体）
- 上游：[[文档切分策略（Chunking）]]（chunk 是检索的基本单元）
- 上游：[[相似度检索]]（Day 4 —— precision@K 评估检索质量）
- 应用：[[QuestionAnswerAdvisor RAG 问答]]（Day 5 —— 手动实现 RAG 三阶段管线）
- 下游：Week 6 混合检索 + rerank（提升检索天花板）
- 下游：Week 7 Agent Tool Calling（RAG 作为 Agent 的一个 Tool）
- 周索引：[[Week5 索引]]
