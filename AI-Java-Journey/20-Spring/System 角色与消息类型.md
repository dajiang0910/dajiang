# System 角色与消息类型

> 给 AI "定规矩"的手段。System 消息设定 AI 的身份和行为规范，是所有消息中优先级最高的。

## 三种消息角色

Chat API 的每次请求包含一个消息数组，每条消息都有"角色"：

| 角色 | 谁说的 | 作用 | 示例 |
|---|---|---|---|
| **system** | 开发者 | 设定 AI 行为规范、人设 | "你是专业翻译助手，只返回译文不加解释" |
| **user** | 最终用户 | 用户的问题/输入 | "把这段文字翻译成英文" |
| **assistant** | LLM | AI 的回复 | "Dependency injection is a design pattern" |

**优先级**：system > user。System 在最前面，对整个对话施加全局约束。

## 为什么需要 System 角色

**没有 system**：AI 自由发挥，输出不可控：
```
用户："翻译成英文：你好"
AI："Hello。这是中文'你好'的英文翻译，意思是..." ← 多余的废话
```

**有 system**：AI 被约束，输出干净：
```
System："你是翻译助手，只返回译文，不加解释"
用户："翻译成英文：你好"
AI："Hello" ← 精准
```

System 本质是**给 AI 戴上"角色面具"**，让它在面具下说话。

## 在 ChatClient 中使用 .system()

```java
// .system() 和 .user() 平级，都在 .prompt() 之后、.call() 之前
chatClient.prompt()
    .system("你是一个专业翻译助手。请将用户文本翻译成英文，只返回译文。")
    .user("依赖注入是一种设计模式")
    .call()
    .content();
// → "Dependency injection is a design pattern."
```

**方法链顺序建议**：`.system()` 放前面（先定规矩），`.user()` 放后面（再给输入）。

## 实战案例：翻译助手

```java
public String translate(String text, String targetLanguage) {
    return chatClient.prompt()
            .system("你是专业翻译助手。请将文本翻译成" + targetLanguage
                    + "。只返回译文，不加解释、注释或额外信息。")
            .user(text)
            .call()
            .content();
}
```

**端点**：`POST /api/chat/translate`
```json
// 请求
{"text": "依赖注入是一种设计模式", "targetLanguage": "英文"}

// 响应
{"code": 200, "data": "Dependency injection is a design pattern."}
```

## System 提示词怎么写

好的 system prompt 包含三要素：

| 要素 | 含义 | 示例 |
|---|---|---|
| **角色** | 你是谁 | "你是资深 Java 开发专家" |
| **行为规范** | 怎么做 | "只返回译文，不加解释" |
| **输出格式** | 输出长什么样 | "纯文本，不要 markdown 格式" |

```java
// ❌ 太模糊
.system("你是一个助手")

// ✅ 明确角色 + 行为 + 格式
.system("你是资深 Java 开发专家。请为代码生成 Javadoc 注释，"
      + "包含 @param、@return、@throws。只返回注释，不修改代码。")
```

## 面试怎么问

- "system/user/assistant 三种角色区别？" → system 设规范（开发者），user 给输入（用户），assistant 是回复（LLM）
- ".system() 有什么用？" → 约束 AI 输出格式、设定人设、提高回复质量
- "system 和 user 消息谁优先级高？" → system，它在对话最前面，全局生效

## 关联

- Day 1 基础：[[Spring AI 起步]] —— 从最简单的 .user() + .call() 开始
- Day 2 组合技：[[PromptTemplate 模板化提示词]] —— system 管"谁来做"，Template 管"做什么"
- 三个端点：`POST /api/chat`（自由对话）、`POST /api/chat/translate`（翻译）、`POST /api/chat/slug`（命名）
- 方法论：system prompt 工程是 Prompt Engineering 的核心之一
