# PromptTemplate 模板化提示词

> 让提示词像 SQL 参数一样，用占位符 `{name}` 代替拼接，可读、可维护、可复用。

## 为什么需要模板

Day 1 的写法——字符串拼接：
```java
// ❌ 难读、容易出错、不可复用
String prompt = "请对以下内容做摘要，不超过 " + maxWords
              + " 字，保留核心要点：\n" + content;
```

PromptTemplate 的写法——占位符 + 填充：
```java
// ✅ 模板清晰、参数命名、可复用
PromptTemplate template = new PromptTemplate(
    "请对以下内容做摘要，不超过 {maxWords} 字，保留核心要点：\n{content}"
);
template.add("maxWords", maxWords);
template.add("content", content);
String prompt = template.render();
```

## 三步操作

```
① 创建模板         ② 填充占位符           ③ 渲染 → 发送
PromptTemplate  →  .add("name", value)  →  .render()  →  .user(rendered)
    ↓                    ↓                     ↓
"{name}你好"         name="张三"          "张三你好"
```

```java
// 完整流程
PromptTemplate template = new PromptTemplate("""
        请对以下内容进行摘要，要求：
        1. 不超过 {maxWords} 字
        2. 保留核心要点，去掉冗余描述
        3. 用中文输出

        内容：
        {content}
        """);

template.add("maxWords", 100);         // 填第一个坑
template.add("content", "长文本...");   // 填第二个坑

String rendered = template.render();   // 渲染成最终提示词

chatClient.prompt()
        .user(rendered)
        .call()
        .content();
```

## 核心 API

| 方法 | 作用 |
|---|---|
| `new PromptTemplate(String)` | 用字符串创建模板 |
| `new PromptTemplate(Resource)` | 从文件加载模板（适合大型提示词） |
| `.add(String, Object)` | 填充一个占位符，可链式调用 |
| `.render()` | 渲染为最终字符串 |
| `.render(Map)` | 一次性传入所有占位符的 Map |
| `.create()` | 直接创建 Spring AI `Prompt` 对象 |
| `.createMessage()` | 渲染为 `Message` 对象（高级用法） |

## PromptTemplate vs String.format

| | PromptTemplate | String.format |
|---|---|---|
| 占位符 | `{name}` 命名 | `%s` / `%d` 位置 |
| 可读性 | ✅ 一眼看出填了什么 | ❌ 需数位置 |
| 模板来源 | 字符串 / 文件（Resource） | 只能字符串 |
| 生态统一 | ✅ 和 Spring AI 一套 | — |
| 进阶能力 | 直接创建 Prompt/Message | 只能渲染字符串 |
| 性能 | 基本无差异 | 基本无差异 |

**一句话**：提示词场景用 PromptTemplate（可读性 + 生态），简单格式化用 String.format。

## 实战案例：摘要生成器

```java
public String summarize(String content, int maxWords) {
    PromptTemplate template = new PromptTemplate("""
            请对以下内容进行摘要，要求：
            1. 不超过 {maxWords} 字
            2. 保留核心要点，去掉冗余描述
            3. 用中文输出

            内容：
            {content}
            """);

    template.add("maxWords", maxWords);
    template.add("content", content);

    return chatClient.prompt()
            .user(template.render())
            .call()
            .content();
}
```

**端点**：`POST /api/chat/summarize`
```json
// 请求
{"content": "Spring AI 是 Spring 生态的 AI 集成框架...（长文本）", "maxWords": 100}

// 响应
{"code": 200, "data": "Spring AI 将 LLM 抽象为普通 Spring 依赖..."}
```

## System + PromptTemplate 结合

两者互补，不是二选一：

```java
chatClient.prompt()
    .system("你是 URL 命名专家...")    // System 管"谁来做"
    .user(template.render())           // Template 管"做什么"
    .call()
    .content();
```

| 维度 | System 角色 | PromptTemplate |
|---|---|---|
| 解决的问题 | **怎么回答**（风格、格式、人设） | **回答什么**（输入参数化） |
| 变化频率 | 很少变（角色固定） | 每次请求都不同 |
| 类比 | 给员工定岗定责 | 给员工派任务单 |

## 面试怎么问

- "为什么用 PromptTemplate 不用 String.format？" → 命名占位符可读性强、支持从文件加载、和 Spring AI 生态统一
- "PromptTemplate 怎么用？" → 创建 → add 填坑 → render 渲染 → 传给 ChatClient
- "和 system 角色怎么配合？" → system 管输出规范，Template 管输入参数化，各司其职

## 关联

- 配套角色约束：[[System 角色与消息类型]] —— system 管"怎么答"，Template 管"答什么"
- Day 1 基础：[[Spring AI 起步]] —— 从手写字符串到模板化的演进
- DTO 模式：[[DTO 与 Entity 之分]] —— SummarizeRequest 同样是 record DTO，统一校验风格
- 后续进阶：Week 4 的 Prompt 工程（JSON Schema 约束输出格式）
