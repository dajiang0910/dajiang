# BeanOutputConverter 结构化输出

> 让 LLM 直接返回 Java Bean，而不是自由文本——告别手写正则提取 JSON 的时代。

## 它解决什么问题

Week 3 的做法——让 LLM 返回结构化数据：
```java
// 脆弱方式：Prompt 里要求返 JSON → 拿 String → 手动 Jackson 解析
String reply = chatClient.prompt()
    .user("请返回 JSON：{\"title\":..., \"keywords\":[...]}")
    .call().content();
// ↓ 各种边界情况：LLM 可能加解释文字、markdown 包裹、字段名不对
NoteMetadata meta = objectMapper.readValue(reply, NoteMetadata.class);
```

Week 4 的做法——一行 `.entity()` 搞定：
```java
NoteMetadata meta = chatClient.prompt()
    .user("请提取标题和关键词：\n" + content)
    .call()
    .entity(NoteMetadata.class);  // ← 自动 Schema → 约束 → 反序列化
meta.getTitle();     // 直接用！
meta.getKeywords();  // 类型安全！
```

## 底层原理：3 步转换链路

```
┌────────────────────────────────────────────────────────────────┐
│  Step 1: 生成 Schema                                          │
│  Java Class (NoteMetadata) → BeanOutputConverter → JSON Schema │
│  {"type":"object","properties":{"title":{"type":"string"},...},│
│   "required":["title","keywords","category"]}                  │
├────────────────────────────────────────────────────────────────┤
│  Step 2: 约束输出                                              │
│  System Prompt + JSON Schema → Prompt → 发给 LLM              │
│  "请按以下 JSON Schema 格式回复：{...schema...}"               │
│  LLM 必须按 Schema 输出，跑偏概率大幅下降                       │
├────────────────────────────────────────────────────────────────┤
│  Step 3: 自动反序列化                                          │
│  LLM 回复（JSON 字符串） → Jackson → Java Bean                 │
│  → NoteMetadata{title="Spring AI", keywords=[...], ...}        │
└────────────────────────────────────────────────────────────────┘
```

## 核心 API：`.entity(Class)`

```java
// Spring AI 2.0 ChatClient 的高层 API
NoteMetadata meta = chatClient.prompt()
        .system("你是一个专业的文档分析专家...")
        .user(content)
        .call()
        .entity(NoteMetadata.class);  // ← 一行替代：手动 Schema + 手动解析
```

对比 `.content()` vs `.entity()`：

| | `.content()` | `.entity(Class)` |
|---|---|---|
| 返回类型 | `String` | `T`（类型安全） |
| 需要手动解析？ | ✅ 要手写 Jackson 代码 | ❌ 自动反序列化 |
| Schema 约束？ | ❌ 只有 Prompt 约束 | ✅ JSON Schema 硬约束 |
| LLM 跑偏风险 | 高（多写解释、markdown 包裹） | 低（Schema 限制了输出格式） |
| 适用场景 | 对话、翻译、摘要 | 提取、分类、实体识别 |

## 底层手动用法：BeanOutputConverter

`.entity()` 的引擎是 `BeanOutputConverter`，了解它有助于调试：

```java
// .entity() 底层实际做的事：
var converter = new BeanOutputConverter<>(NoteMetadata.class);
String jsonSchema = converter.getJsonSchema();  // 查看生成的 Schema

// 手动调用（需要更多控制的场景）：
var response = chatClient.prompt()
        .user(content)
        .call()
        .chatClientResponse();
String text = response.chatResponse().getResult().getOutput().getText();
NoteMetadata result = converter.convert(text);  // 手动转换
```

## 约束字段：`@JsonProperty(required = true)`

```java
public record NoteMetadata(
    @JsonProperty(required = true, value = "title")  // ← 必填
    String title,

    @JsonProperty(required = true, value = "keywords")
    List<String> keywords,

    String difficulty  // ← 可选，LLM 可以不给
) {}
```

`required = true` → JSON Schema 里加入 `"required": ["title", "keywords"]` → LLM 必须返回这些字段。

## 防跑偏策略

| 策略 | 说明 | 优先级 |
|---|---|---|
| **@JsonProperty(required=true)** | Schema 层面强制约束字段存在 | ⭐⭐⭐ |
| **System 角色约束** | "只返回 JSON，不要加任何解释或 markdown 标记" | ⭐⭐⭐ |
| **temperature=0** | 降低随机性，让输出更稳定 | ⭐⭐ |
| **Few-shot 示例** | 给 LLM 1-2 个正确示例 | ⭐⭐ |
| **Fallback 解析** | entity() 失败 → 正则提取 JSON → 再解析 | ⭐ |

## 代码实战

### NoteMetadata —— 结构化输出的"合同"

```java
public record NoteMetadata(
    @JsonProperty(required = true, value = "title") String title,
    @JsonProperty(required = true, value = "keywords") List<String> keywords,
    @JsonProperty(required = true, value = "category") String category,
    String difficulty,
    String summary
) {}
```

### StructuredExtractService —— 一行 entity() 完成提取

```java
@Service
public class StructuredExtractService {
    private final ChatClient chatClient;

    public StructuredExtractService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public NoteMetadata extract(String content) {
        return chatClient.prompt()
                .system("""
                        你是一个专业的文档分析专家...
                        1. title — 内容的核心标题
                        2. keywords — 3-5 个关键词
                        3. category — 技术/管理/产品/设计/其他
                        重要：只返回 JSON，不要加任何解释或 markdown 代码块标记。
                        """)
                .user(content)
                .call()
                .entity(NoteMetadata.class);  // ← 核心 API
    }
}
```

### Controller 端点

```java
@PostMapping("/extract/metadata")
public ApiResponse<NoteMetadata> extractMetadata(@Valid @RequestBody ExtractRequest request) {
    return ApiResponse.success(extractService.extract(request.content()));
}
```

### curl 验收

```bash
curl -X POST http://localhost:8080/api/extract/metadata \
  -H "Content-Type: application/json" \
  -d '{"content": "Spring AI 是 Spring 生态的 AI 框架..."}'

# 返回：
# {"code":200,"data":{"title":"Spring AI 框架介绍","keywords":["Spring AI","LLM"],"category":"技术",...}}
```

## 面试怎么问

- "怎么让 LLM 返回结构化数据？" → BeanOutputConverter：Java Class → JSON Schema → 约束 Prompt → 自动反序列化。`.entity(Class)` 一行搞定。
- "LLM 不按 Schema 返回怎么办？" → 四层防跑偏：① `@JsonProperty(required=true)` 硬约束 ② System 角色限制输出格式 ③ `temperature=0` 降低随机性 ④ Fallback：手动正则提取 JSON 兜底
- "`.entity()` 和 `.content()` + 手动 Jackson 的区别？" → `.entity()` 是主动约束（Schema 让 LLM 不易跑偏），手动是被动补救。前者从根本上减少了解析失败率。

## 关联

- 前置：[[Spring AI 起步]] —— ChatClient 基础，`.entity()` 是 `.call()` 之后的新取值方法
- 前置：[[依赖注入(DI)]] —— StructuredExtractService 同样用 ChatClient.Builder 构造器注入
- 前置：[[统一响应体（ApiResponse）]] —— `ApiResponse<NoteMetadata>` 泛型复用验证
- 前置：[[@RestController]] —— 新增 `POST /api/extract/metadata` 端点
- 前置：[[DTO 与 Entity 之分]] —— NoteMetadata 是输出 DTO，ExtractRequest 是输入 DTO
- 配套：[[Prompt 工程进阶]] —— 少样本 + 约束输出 + 防幻觉，结构化输出的 Prompt 侧
- 后续：[[Apache Tika 文档解析]] —— 文档文本抽取 → 结构化提取，形成完整接入链路
