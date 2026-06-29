# Prompt 工程进阶（Few-shot + Schema 约束）

> Day 1 解决了"有 Schema"；Day 2 解决"Schema 里的字段填得准"。Few-shot 范例 > 纯文字描述，LLM 是模式补全器不是指令执行器。

## 它解决什么问题

Day 1 的 `BeanOutputConverter` 会自动生成 JSON Schema 发给 LLM，但 Schema 只定义了字段名和类型，**不定义每个字段该填什么值**。结果：
- `category` 要求"技术/管理/产品/设计/其他"，LLM 可能返回"编程"
- `difficulty` 要求"入门/中级/高级"，LLM 可能返回"简单"
- `keywords` 可能只返回 1 个，不在预期的 3-5 个范围

Day 2 用三层增强解决这个问题：

```
v1 基础版（Day 1）:
  System Prompt「提取标题/关键词/分类」→ JSON Schema 约束结构 → .entity()

v2 增强版（Day 2）: 
  Few-shot 范例「输入 → 期望输出」× 2 
  + @JsonPropertyDescription 字段语义描述 
  + 后校验（category 枚举兜底）
  → 增强后的 Prompt + 增强 Schema → .entity() → validateAndFallback()
```

## 核心要点

### 1. Few-shot 为什么有效？

LLM 本质上是 **next-token predictor（下一个 token 预测器）**，不是指令执行器：
- **文字描述**：LLM 要"理解→推理→执行"，链路长，偏差大
- **Few-shot 范例**：LLM 直接"模式匹配→照做"，链路短，准确率高

```
❌ 低效：用 200 字描述格式
「category 必须是技术/管理/产品/设计/其他之一，difficulty 可选值为入门/中级/高级...」

✅ 高效：给 2 个完整范例
「范例1：输入"Spring AI 是 Spring 生态..." → 输出{"title":"Spring AI 介绍",...}
  范例2：输入"Q3 发布计划..." → 输出{"title":"Q3 发布计划",...}」
```

**工程经验**：Few-shot 是提升结构化输出质量的最有效单一手段。

### 2. @JsonPropertyDescription —— 给字段写"语义说明"

```java
// Day 1：只定义结构，不定义语义
public record NoteMetadata(
    @JsonProperty(required = true) String title,
    @JsonProperty(required = true) List<String> keywords,
    @JsonProperty(required = true) String category
) {}

// Day 2：每个字段带上语义描述 → 写入 JSON Schema
public record NoteMetadata(
    @JsonProperty(required = true)
    @JsonPropertyDescription("内容的核心标题，不超过 50 字，简洁概括主题")  // ← 新增！
    String title,

    @JsonProperty(required = true)
    @JsonPropertyDescription("3-5 个关键词，用于检索和分类")
    List<String> keywords,

    @JsonProperty(required = true)
    @JsonPropertyDescription("内容分类，必须是以下之一：技术、管理、产品、设计、其他")
    String category
) {}
```

效果对比——生成的 JSON Schema：

```json
// Day 1 Schema（只定义类型）
{"properties": {"category": {"type": "string"}}}

// Day 2 Schema（类型 + 语义描述）
{"properties": {
  "category": {
    "type": "string",
    "description": "内容分类，必须是以下之一：技术、管理、产品、设计、其他"
  }
}}
```

LLM 看到 `description` 后，跑偏概率显著下降。

### 3. 五层防跑偏策略（Day 1 + Day 2 合并）

| 层级 | 策略 | 作用 | 新增于 |
|------|------|------|--------|
| **L1** | `@JsonProperty(required=true)` | Schema 层面强制字段存在 | Day 1 |
| **L2** | `@JsonPropertyDescription` | Schema 层面给每个字段加语义说明 | Day 2 🆕 |
| **L3** | Few-shot 范例 | Prompt 层面给 LLM 看"标准答案" | Day 2 🆕 |
| **L4** | System 角色约束 | Prompt 层面限制输出格式 | Day 1 |
| **L5** | 后校验 `validateAndFallback()` | 代码层面兜底修正 | Day 2 🆕 |

> 生产实战经验：L1-L4 是"预防"，L5 是"兜底"。预防降低概率，兜底确保下限。

### 4. Prompt 外部化

当前 System Prompt 硬编码在 Java 代码中。后续可抽取到 `src/main/resources/prompts/` 目录，用 `@Value` 或 `Resource` 加载：

```java
// 未来优化方向：
@Value("classpath:prompts/extract-metadata.st")
private Resource promptTemplate;
```

这样做的好处：① 非技术人员可直接改 Prompt 不需重新编译 ② 不同环境可用不同 Prompt ③ A/B 测试更容易。

## Java 里怎么落地

### StructuredExtractService.extractV2()

```java
public NoteMetadata extractV2(String content) {
    NoteMetadata result = chatClient.prompt()
            .system("""
                    你是一个专业的文档分析专家...
                    
                    ——— 范例 1 —— 输入：
                    Spring AI 是 Spring 生态的 AI 框架...
                    → 期望输出：
                    {"title": "Spring AI 框架介绍", "keywords": [...], "category": "技术", ...}
                    
                    ——— 范例 2 —— 输入：
                    企业微信 Q3 发布计划...
                    → 期望输出：
                    {"title": "企业微信 Q3 发布计划", "keywords": [...], "category": "产品", ...}
                    
                    现在请处理下面的输入：
                    """)
            .user(content)
            .call()
            .entity(NoteMetadata.class);

    return validateAndFallback(result);  // 后校验兜底
}
```

### 后校验逻辑

```java
private NoteMetadata validateAndFallback(NoteMetadata raw) {
    var validCategories = Set.of("技术", "管理", "产品", "设计", "其他");
    
    if (!validCategories.contains(raw.category())) {
        // 模糊匹配 → 否则归为"其他"
        String fixed = switch (raw.category()) {
            case String c when c.contains("技术") || c.contains("编程") -> "技术";
            case String c when c.contains("产品") || c.contains("功能") -> "产品";
            // ...
            default -> "其他";
        };
        return new NoteMetadata(raw.title(), raw.keywords(), fixed, ...);
    }
    return raw;
}
```

## 面试怎么问

- **"怎么提高结构化输出的准确率？"** → ① Few-shot 范例（最有效，LLM 是模式补全器）② `@JsonPropertyDescription` 字段语义描述写入 Schema ③ System Prompt 中枚举约束 + 格式限制 ④ 后校验兜底（模糊匹配修正 + 重试机制）

- **"Few-shot 和 Zero-shot 的区别？"** → Zero-shot 只给指令不给范例；Few-shot 给 2-3 个范例让 LLM "照猫画虎"。对于结构化提取，Few-shot 的准确率提升 > 20%。成本 = 多消耗 ~200-500 input token。

- **"@JsonPropertyDescription 和 @Schema 有什么区别？"** → `@JsonPropertyDescription`（Jackson）影响 JSON Schema 的 `description` 字段，LLM 能看到；`@Schema`（Swagger）只影响 OpenAPI 文档，LLM 看不到。

## 关联

- 前置：[[BeanOutputConverter 结构化输出]]（Day 1 —— `.entity()` 基础用法，3 步转换链路）
- 前置：[[Spring AI 起步]] —— ChatClient 基础
- 前置：[[System 角色与消息类型]] —— System Prompt 是 Few-shot 范例的载体
- 前置：[[PromptTemplate 模板化提示词]] —— Prompt 模板化，后续可外部化
- 后续：[[Apache Tika 文档解析]]（Day 3）—— 文档 → 文本 → 结构化提取，完整接入链路
- 反方向：[[超时重试与Token成本]] —— Few-shot 增加 token 消耗，需要成本意识
