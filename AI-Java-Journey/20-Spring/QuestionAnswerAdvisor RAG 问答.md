# QuestionAnswerAdvisor RAG 问答

> 一句话：Day 5 的核心是手动实现 RAG 三阶段管线——不是调用框架的自动 Advisor，而是自己写 Retrieve → Augment → Generate，精确控制每一步的行为。

## 它解决什么问题

Day 1-4 分别解决了"向量化"、"向量存储"、"文档切分灌库"、"语义检索"四个独立环节。Day 5 把它们**串成一条完整链路**：用户提问 → 自动检索知识库 → 把检索结果注入 Prompt → LLM 基于知识生成答案。

**用框架 Advisor vs 手动实现**：Spring AI 1.x 有 `QuestionAnswerAdvisor` 自动完成检索+增强，但 2.0.0 已移除它。手动实现虽然多写几十行代码，但获得了精确控制——什么时候检索、检索多少条、Prompt 怎么拼、无结果怎么兜底，全由你掌控。

## 核心要点

### 1. RAG 三阶段数据流（Day 5 实现）

```
RagController.ask()                      // HTTP 层
  → RagService.ask(question, topK, threshold)
      → [1] SearchService.searchAndFormat()     // 检索层
            → VectorStore.similaritySearch()     // Redis HNSW KNN
            → 格式化为 formattedText（可拼 Prompt 的文本块）
      → [2] buildSystemPrompt(searchResult)      // 增强层
            → 有结果：注入知识片段 + 行为约束
            → 无结果：强硬兜底语，禁止编造
      → [3] ChatClient.prompt().call()           // 生成层
            → System = 增强后的 Prompt
            → User = 用户原始问题
            → 返回带引用的答案
      → RagResponse（question + answer + sources + elapsedMs）
```

### 2. buildSystemPrompt 的分支设计（防幻觉核心）

```java
if (检索结果为空) {
    → "你是企业知识库智能助手。当前知识库中未检索到相关信息。
       请如实告知用户，不要编造任何信息。"
    // 关键词：如实告知 + 不要编造 → 强制 LLM 走兜底路径
} else {
    → "严格基于以下知识片段回答...引用具体片段编号...
       不要编造任何知识片段中没有的信息"
    // 关键词：严格基于 + 引用片段 + 不要编造 → 把 LLM 锁在知识库内
}
```

**为什么区分很重要？** 如果不区分，检索结果为空时拼入空字符串，LLM 会认为"没有约束"从而自由发挥——开始用训练数据瞎编。区分后，空检索 → LLM 只能回复"暂无相关信息"。

### 3. System Prompt vs User Prompt（安全防线）

```
✅ 正确做法：
  System Prompt = 角色设定 + 知识片段 + 行为约束
  User Prompt   = 用户原始问题

❌ 错误做法：
  System Prompt = 角色设定
  User Prompt   = 知识片段 + 用户问题  ← 用户可注入"忽略以上资料"
```

**面试加分点**：System Prompt 在 Transformer 注意力机制中具有更高优先级，模型倾向于服从系统指令。知识片段放入 System Prompt 是 RAG 的第一道安全防线。

### 4. RAG 的"安全失败"原则

```
检索成功 → 基于知识回答（有据可查）
检索失败 → 明确告知"暂无相关信息"（安全失败）
       → ❌ 不应降级为"用训练数据回答"（破坏了 RAG 的可信性）
```

这是一个工程决策：企业知识库场景优先保证"宁可不答，不可乱答"。

### 5. 三个端点的设计意图

| 端点 | 用途 | 教学价值 |
|------|------|---------|
| `POST /api/qa/ask` | 标准 RAG 问答 | 生产端点 |
| `POST /api/qa/ask-without-rag` | 纯 LLM 问答 | 对比 baseline |
| `POST /api/qa/compare` | 同时返回两者 | 直观展示 RAG 价值 |

## Java 里怎么落地

### RagService：管线指挥官

```java
@Service
public class RagService {
    private final SearchService searchService;   // 检索层
    private final ChatClient chatClient;          // 生成层

    public RagResponse ask(String question, int topK, double threshold) {
        // Phase 1: Retrieve
        SearchResult ctx = searchService.searchAndFormat(question, topK, threshold);

        // Phase 2: Augment
        String systemPrompt = buildSystemPrompt(ctx);  // 区分有/无结果

        // Phase 3: Generate
        String answer = chatClient.prompt()
                .system(systemPrompt)
                .user(question)
                .call()
                .content();

        return new RagResponse(question, topK, threshold, ctx.hits(), answer, elapsed);
    }
}
```

### RagController：HTTP 入口

- `POST /api/qa/ask` — `{question, topK, threshold}` → `{answer, sources, hitCount, elapsedMs}`
- `POST /api/qa/compare` — 同上，但返回 `{rag: {...}, pureLlm: {...}}`
- `POST /api/qa/ask-without-rag` — 纯 LLM，对比用

### curl 测试

```bash
# 标准 RAG 问答
curl -X POST http://localhost:8080/api/qa/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "如何退款", "topK": 4, "threshold": 0.6}'

# RAG vs 纯 LLM 对比
curl -X POST http://localhost:8080/api/qa/compare \
  -H "Content-Type: application/json" \
  -d '{"question": "如何退款", "topK": 4, "threshold": 0.6}'
```

## 踩过的坑 / 取舍

### 取舍：手动实现 vs 框架 Advisor

| | Spring AI 1.x QuestionAnswerAdvisor | Day 5 手动实现 |
|---|---|---|
| 代码量 | 2 行配置 | ~100 行 |
| 检索控制 | 框架默认 | 完全自定义 topK/threshold |
| Prompt 控制 | 框架模板 | 精确控制每句话 |
| 空结果兜底 | 框架默认语 | 自定义兜底话术 |
| 异常处理 | 黑盒 | 透明可控 |
| 面试能讲透 | ❌ "调了个 API" | ✅ "我实现了三阶段管线" |

**选择手动实现的原因**：Spring AI 2.0.0 已移除 QuestionAnswerAdvisor；即使存在，手动实现的内化效果远超调用框架 API。

### Spring AI 2.0.0 版本注意

- `QuestionAnswerAdvisor` 在 2.0.0 中被移除，Advisor API 重构为 `CallAdvisor` / `StreamAdvisor` 接口
- 原来的 `VectorStore` 检索 + Prompt 注入逻辑需要手动编排
- 这对学习是好事——必须理解 RAG 底层才能实现

## 面试怎么问

> **"你们的 RAG 管线是怎么实现的？为什么不用 LangChain 的链式调用？"**

**答题思路**（STAR 格式）：

- **S**：项目需要企业知识库问答，回答须可追溯来源、不能幻觉
- **T**：初期试用 Spring AI 1.x 的 QuestionAnswerAdvisor，发现自动挡带来三个问题——Prompt 模板不可控（无法定制兜底话术）、检索参数被框架默认值覆盖、空结果时 LLM 仍然自由编造
- **A**：自研 RAG 三阶段管线：检索层用 VectorStore.similaritySearch + 后置过滤（Day 4 踩坑）、增强层手动 buildSystemPrompt 区分有无结果、生成层 ChatClient 调用。关键是 System Prompt 放知识片段（防注入）、无结果分支强约束（防幻觉）
- **R**：提示词注入风险消除、空检索幻觉下降、Token 可控、可灵活叠加限流和日志；Spring AI 2.0.0 也刚好移除了这个 Advisor，验证了"手动挡"更符合企业场景

> **"检索没结果时，你们是直接告诉用户'不知道'，还是降级到纯 LLM 回答？"**

企业知识库场景选择"安全失败"——明确告知暂无相关信息 + 引导用户换关键词或联系管理员。不降级到纯 LLM 的理由：一旦 LLM 在 RAG 模式下开始用训练数据回答，用户就无法区分"这是知识库里的"还是"LLM 编的"，破坏了可信性。

## 关联

- 核心理论：[[RAG 原理与最小闭环]]（三阶段管线原理 + 局限性）
- 上游：[[相似度检索]]（Day 4 —— SearchService 是 RagService 的检索层）
- 上游：[[Redis Stack 向量存储]]（检索的物理载体）
- 上游：[[Embedding 向量化]]（查询向量化的入口）
- 上游：[[文档切分策略（Chunking）]]（chunk 质量决定检索质量）
- 下游：Week 6 混合检索 + rerank（当检索质量不够时，提升检索天花板）
- 下游：Week 7 Agent Tool Calling（RAG 作为 Agent 的一个 Tool）
- 项目：[[Week5 索引]]

## 广告

> 本项目（`notes-api`）的 `RagService` 和 `RagController` 在 Spring AI 2.0.0 + 阿里百炼上验证通过，完整代码见 `D:\User\tangzg\java-ai-journey\notes-api`。
