# Redis Stack 向量存储

> 一句话：Redis Stack = 普通 Redis + RediSearch 向量模块，让 Redis 从"缓存引擎"升级为"向量数据库"，支持 KNN 语义检索。

## 它解决什么问题

RAG 的核心环节：用户问题 → 向量化 → **在向量库里找最相似的文档片段** → 拼入 Prompt → LLM 生成答案。

普通 Redis 只有 key-value 存储能力，没有"按语义相似度找最近邻"的能力。你需要的是：存向量 → 按余弦相似度搜 top-K → 返回原文。**Redis Stack 预装了 RediSearch 模块，提供了 HNSW 索引 + KNN 检索，一步到位。**

不用它会怎样？如果裸写：把所有向量加载到内存 → 逐条算余弦相似度 → 排序 → 取 top-K，这叫**暴力检索（O(N)）**。1000 条还能忍，10 万条直接超时。

## 核心要点

### 1. VectorStore = 向量数据库的 JDBC

```java
// 不管背后是 Redis Stack、PGVector 还是 Milvus，这三行一模一样：
vectorStore.add(List.of(document));                    // 灌入
List<Document> results = vectorStore.similaritySearch(  // 检索
    SearchRequest.query("如何退款").topK(3).similarityThreshold(0.7));
```

| JDBC 概念 | VectorStore 对应 |
|-----------|-----------------|
| `DataSource` | Redis/Jedis 连接配置 |
| `JdbcTemplate` | `VectorStore` 接口 |
| `INSERT` | `vectorStore.add(docs)` |
| `SELECT ... WHERE` | `vectorStore.similaritySearch(query)` |
| 换数据库改 URL | 换 `application.properties` + Maven 依赖 |

### 2. Redis Stack vs 普通 Redis

| | 普通 Redis | Redis Stack |
|---|---|---|
| 向量检索 | ❌ 无 | ✅ RediSearch KNN |
| 元数据过滤 | 手动维护 | ✅ 内置 tag/numeric/text 字段 |
| 全文检索 | ❌ | ✅ FT.SEARCH |
| 混合检索 | ❌ | ✅ 向量 + 全文 + 元数据组合查询 |

### 3. HNSW 索引算法（面试高频）

**HNSW**（Hierarchical Navigable Small World）= 分层可导航小世界图。

关键参数：
- **M**（max connections）：每个节点连多少邻居，越大越精确但索引越大。默认 16
- **efConstruction**：建索引时搜索深度，越大索引质量越高但建得越慢。默认 200
- **efRuntime**：查询时搜索深度，越大越精确但越慢。默认 10

直觉理解：HNSW 就像"高速公路 + 乡间小路"——顶层是高速公路（快速定位区域），底层是乡间小路（精细搜索）。查询时从顶层一路往下，每层只搜几个邻居，时间复杂度 O(log N)。

### 4. RedisVectorStore 自动配置

Spring AI 2.0.0 的 `RedisVectorStoreAutoConfiguration` 自动创建 `RedisVectorStore` Bean，条件是：
- `EmbeddingModel` Bean 存在（✅ 已有）
- `JedisConnectionFactory` Bean 存在（✅ `spring-boot-starter-data-redis` + Jedis）
- `spring-ai-starter-vector-store-redis` 在 classpath

配置速查：
```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379

spring.ai.vectorstore.redis.index-name=notes-knowledge
spring.ai.vectorstore.redis.prefix=doc:
spring.ai.vectorstore.redis.initialize-schema=true

spring.ai.vectorstore.redis.hnsw.m=16
spring.ai.vectorstore.redis.hnsw.ef-construction=200
spring.ai.vectorstore.redis.hnsw.ef-runtime=10
```

### 5. `similaritySearch(String)` vs `similaritySearch(SearchRequest)`

```java
// 快捷方式：只传查询文本，topK 和 threshold 用默认值
vectorStore.similaritySearch("如何退款");

// 完整控制：指定 topK、最低相似度阈值、元数据过滤
vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("如何退款")
        .topK(5)
        .similarityThreshold(0.7)
        .filterExpression("source == 'help-center'")  // 元数据过滤
        .build()
);
```

## Java 里怎么落地

本项目（Spring AI 2.0.0）：

### Docker Compose

```yaml
services:
  redis-stack:
    image: redis/redis-stack:latest
    container_name: notes-redis-stack
    ports:
      - "6379:6379"
      - "8001:8001"  # RedisInsight 管理界面
    volumes:
      - redis_data:/data
    restart: unless-stopped
```

### Maven 依赖

```xml
<!-- Spring AI Redis Vector Store 启动器 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-redis</artifactId>
    <version>2.0.0</version>
</dependency>
<!-- Spring Data Redis + Jedis 客户端 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

### VectorStoreService 核心代码

```java
@Service
public class VectorStoreService {
    private final VectorStore vectorStore;

    public VectorStoreService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    // 灌入单文档
    public void ingest(String content, Map<String, Object> metadata) {
        Document doc = new Document(content, metadata);
        vectorStore.add(List.of(doc));
    }

    // 语义检索
    public List<Document> search(String query, int topK, double threshold) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query).topK(topK)
                .similarityThreshold(threshold).build());
    }
}
```

### REST 端点

- `POST /api/vector/ingest`：灌入文档到向量库
- `POST /api/vector/search`：语义检索

## 踩过的坑 / 取舍

### 踩坑 1：Windows 原生 Redis 端口冲突

**现象**：应用启动报 `ERR unknown command 'FT._LIST'`，但 Docker Redis Stack 中 `FT._LIST` 正常。

**根因**：Windows 宿主机的 `redis-server.exe` 先占用了 6379 端口，Java 应用连到了它（无 RediSearch 模块）。

**排查命令**：
```bash
netstat -ano | findstr :6379     # 列出占端口的进程 PID
tasklist | findstr <PID>         # 查进程名
```

**解决**：`taskkill /F /PID <PID>` 杀 Windows 原生 Redis，让 Docker 独占 6379。

### 踩坑 2：密码认证与 Docker Desktop 网络层

Docker Desktop on Windows 的端口转发层有时会导致 AUTH 命令发送异常。本地开发建议不设密码，生产环境用 `--requirepass`。

### 取舍：Redis Stack vs Milvus

| | Redis Stack | Milvus |
|---|---|---|
| 部署 | `docker compose up -d` | 复杂度高（etcd + MinIO + Milvus） |
| 检索性能 | 中等（HNSW，十万级够用） | 强（多种索引，百万/千万级） |
| 混合检索 | ✅ 向量+全文+元数据 | ✅ 更强 |
| v1 阶段 | ✅ 够用 | 过度工程化 |

**当前选择 Redis Stack，W6 过渡到 Milvus。**

## 面试怎么问

> **"你们项目用 Redis 做向量存储，为什么选 Redis Stack？"**

答题要点：
1. Redis Stack 内置 RediSearch 模块，提供 HNSW 索引 + KNN 向量检索
2. 相比普通 Redis：支持向量+全文+元数据混合检索
3. 相比 Milvus：部署简单（一个 `docker compose up`），v1 阶段够用
4. VectorStore 抽象让后续升级到 Milvus 只改配置和依赖，业务代码不动

> **追问："HNSW 是什么？它的三个参数 M / efConstruction / efRuntime 分别控制什么？"**

答题要点：HNSW = 分层可导航小世界图；M = 每节点邻居数（索引大小）；efConstruction = 建索引搜索深度（索引质量）；efRuntime = 查询搜索深度（查询精度）。三者权衡的是"索引大小→建索时间→查询精度"三角。

## 关联

- 上位：[[Embedding 向量化]]（向量从哪来 → 向量存到哪）
- 上位：[[RAG 原理与最小闭环]]（向量存储是检索层的物理载体）
- 应用：[[文档切分策略（Chunking）]]（Day 3 —— 文档切好再灌入）
- 应用：[[相似度检索]]（Day 4 —— 从 Redis Stack 搜出来）
- 支撑：[[QuestionAnswerAdvisor RAG 问答]]（Day 5 —— Advisor 自动检索）
- 降级：SimpleVectorStore（内存实现，Docker 不可用时的逃生舱）
- 周索引：[[Week5 索引]]
