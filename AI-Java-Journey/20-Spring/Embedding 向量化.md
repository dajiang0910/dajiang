# Embedding 向量化

> 一句话：Embedding 是"文字 → 数字"的翻译机——把任意文本映射到高维语义空间中的一个坐标点，语义相近的文本点也离得近。

## 它解决什么问题

LLM 只认识数字，不认识文字。要让模型判断「"猫吃鱼"和"狗吃肉"哪个更像？」，或者要在 1000 篇文档里找到和用户问题最相关的 3 段，**必须先有一套把文字变成可计算的数字表示的方法**。Embedding 就是这套方法——它把文本变成 `float[]`（固定维度的浮点数组），之后的一切检索、聚类、分类都变成向量运算。

**不用它会怎样？** 只能用关键词匹配（BM25/Elasticsearch），搜"自行车"找不到"脚踏车"，搜"怎么退钱"找不到"退款流程"——同义词、口语化表达完全失效。

## 核心要点

### 1. Embedding 是训练出来的，不是人工设计的

Embedding 模型在海量文本上训练，学会了"让语义相近的文本产生相近的向量"。这个"相近"不是人告诉它的，是模型从大量共现模式中学到的。

### 2. 余弦相似度：衡量"语义有多近"的标准工具

```
cos(θ) = (A·B) / (|A| × |B|)
```

- 返回值范围 `[-1, 1]`，越接近 1 越相似
- **为什么不用欧氏距离？** 因为 Embedding 的语义信息编码在**方向**上，而非长度上。两个向量方向几乎一致但长度差几倍 → 语义基本一样，欧氏距离却会说"差很大"。余弦只看方向不看长度，完美适配。

### 3. 维度 = 语义表达力 vs 成本的权衡

| 维度 | 表达力 | 存储 | 检索速度 | 典型模型 |
|------|--------|------|----------|----------|
| 256 | 弱——细节丢得多 | 小 | 快 | 轻量场景 |
| 768 | 中 | 中 | 中 | text-embedding-ada-002 |
| 1536 | 强——细粒度语义 | 较大 | 中 | text-embedding-v2 / text-embedding-3-small |
| 3072 | 最强 | 大 | 慢 | text-embedding-3-large |

**维度是模型架构决定的，不是你调用时可以随便调的参数。** 选模型 = 选维度。1536 是当前中文 Embedding 的行业均衡点。

### 4. 单条 `embed()` vs 完整 `call()` 的真正区别

```java
// 便捷入口：一行拿向量
float[] vec = embeddingModel.embed("猫吃鱼");

// 完整入口：传 EmbeddingOptions 做精细控制
EmbeddingResponse response = embeddingModel.call(
    new EmbeddingRequest(List.of("猫吃鱼", "狗吃肉"),
        EmbeddingOptions.builder()
            .model("text-embedding-v2")
            .user("userId-123")  // 监控/计费分流
            .build())
);
```

**不是"单个 vs 批量"**——`embed(List<String>)` 也能批量。区别是 `call()` 可以传 `EmbeddingOptions`（指定模型版本、user 标识等），是生产环境的标准入口。

### 5. 对相同文本，同一模型每次返回完全一致的向量

Embedding 是**确定性**的（temperature=0 的效果），不涉及随机采样。同一个模型对同一段文本，每次返回的向量一模一样。

## Java 里怎么落地

本项目（Spring AI 2.0.0）：

```java
@Service
public class EmbeddingService {
    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    // 单文本 Embedding
    public float[] embed(String text) {
        return embeddingModel.embed(text);
    }

    // 批量 Embedding（灌库时用）
    public List<float[]> embedBatch(List<String> texts) {
        return embeddingModel.embed(texts);
    }

    // 余弦相似度（8 行手写，比调库更清楚底层在算什么）
    public static double cosineSimilarity(float[] a, float[] b) {
        double dotProduct = 0.0, normA = 0.0, normB = 0.0;
        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

Python 对照（你会在 Python 的 RAG 代码中看到）：
```python
from openai import OpenAI
client = OpenAI()
response = client.embeddings.create(
    model="text-embedding-v2",
    input="猫吃鱼"
)
vector = response.data[0].embedding  # → list[float]
```

### 配置（application.properties）

```properties
# 百炼 text-embedding-v2（1536 维，中文效果好）
spring.ai.openai.embedding.options.model=text-embedding-v2
spring.ai.openai.embedding.options.timeout=10s
```

**同样的 base-url 和 api-key 同时服务 Chat 和 Embedding**——Spring AI 自动复用 `spring.ai.openai.*` 配置。

## 面试怎么问

> **"Embedding 是什么？为什么 RAG 用它做检索？"**

答题要点：
1. Embedding = 文本在高维语义空间中的坐标表示，语义相近 → 向量相近
2. 向量相近 → 余弦相似度高 → 检索时找 top-K 最相似的文档片段
3. 相比关键词检索（BM25），Embedding 能捕获同义词、口语化表达、跨语言语义——搜"自行车"也能命中"脚踏车"
4. 这是 RAG 检索层的数学基础：**用户问题 → 向量 → 最近邻搜索 → 相关片段 → 拼入 Prompt → LLM 生成答案**

> **追问："1536 维是什么？为什么不是 10000 维？"**

答题要点：维度由模型架构决定，不是可调参数。1536 是性能/成本均衡——维度越高表达力越强，但存储和检索成本线性增长。生产环境选模型 = 选维度 = 选成本。

## 踩过的坑 / 取舍

- **百炼 Embedding 模型需要单独开通**：Chat 模型开通不代表 Embedding 模型也开通，要去百炼控制台确认 `text-embedding-v2` 已开通，否则 Spring AI 启动不报错但首次调用抛 404
- **`embed()` vs `call()` 不是"单个 vs 批量"**：Day 1 自测 Q2 的矫正——真正区别是是否传 `EmbeddingOptions`。面试时这个问题答歪直接扣分

## 关联

- 上位：[[RAG 原理与最小闭环]]（Embedding 是 RAG 检索层的数学基础）
- 应用：[[Redis Stack 向量存储]]（Day 2 —— 把向量存起来、搜出来）
- 应用：[[文档切分策略（Chunking）]]（Day 3 —— 把文档切好再 Embedding）
- 支撑：[[相似度检索]]（Day 4 —— 用户问题 → 向量 → 找最近邻）
- 底层：[[QuestionAnswerAdvisor RAG 问答]]（Day 5 —— Advisor 自动做检索+增强）
- 周索引：[[Week5 索引]]
