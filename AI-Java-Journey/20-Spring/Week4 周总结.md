# Week 4 周总结

> **结构化输出 + 文档解析**。Phase 2 正式启动——企业知识库智能助手的"接入层"完工。从"能调 LLM"升级到"让 LLM 返回类型安全的 Java Bean + 多格式文档自动解析 + LLM 调用可观测"。

## 本周成果一览

| 维度 | 数量 | 明细 |
|------|------|------|
| 新增端点 | 5 | `/extract/metadata`、`/extract/metadata/v2`、`/documents/parse`、`/documents/parse-and-extract`、`/documents/analyze` |
| 新增 Service | 3 | `StructuredExtractService`、`DocumentParseService`、`DocumentAnalysisService` |
| 新增 DTO | 4 | `NoteMetadata`、`DocumentParseResponse`、`TextStatistics`、`DocumentAnalysisResponse` |
| 新增 Advisor | 1 | `MetricsAdvisor`（自定义 CallAdvisor → Micrometer） |
| 端点总计 | 18 | W1-2: 5, W3: 8, W4: +5 |
| 测试总计 | **56** | W1-3: 41, W4: +15（全绿 ✅） |
| 本周笔记 | **8 篇** | D3×1, D4×2, D5×2, D6×2, D7×1（本文） |
| 笔记总计 | **70 篇** | 12 周总进度 |

## 五天能力阶梯

```
Day 1: BeanOutputConverter 入门
       .entity(Class) → LLM 直接返回 Java Bean
       Schema → Prompt 约束 → Jackson 反序列化
       四层防跑偏（required → System → temperature=0 → Fallback）

Day 2: Prompt 工程进阶
       Few-shot 范例 + @JsonPropertyDescription + 后校验
       第五层防跑偏（validateAndFallback）
       核心认知：LLM 是模式补全器，范例 > 文字指令

Day 3: Apache Tika 文档解析
       Tika = "文档解析的 JDBC"，一行代码解析 1000+ 格式
       核心坑：detect() 和 parseToString() 不能串联

Day 4: 结构化抽取完整链路
       端到端闭环：MultipartFile → Tika → extractV2 → JSON
       工程化五层防护 + Swagger 文件上传 + 全局异常
       类名冲突坑：ApiResponse vs @ApiResponse（Java 无 import alias）

Day 5: 文档智能分析综合实战
       四段管线：Tika → 统计（程序） → AI 元数据 → AI 关键句
       SimpleLoggerAdvisor = ChatClient 的"中间件"
       成本分界线：能程序算的绝不调 AI

Day 6: 周末整合
       自定义 MetricsAdvisor → Micrometer 指标采集
       错题盲区全部打掉（ApiResponse 冲突、自定义 Advisor、可观测三支柱）
```

## 自测成绩趋势

```
92 ┤  ● D1: BeanOutputConverter 入门
83 ┤      ● D2: Prompt 工程进阶
76 ┤          ● D3: Apache Tika
   │
   │              ● D4: 59  ← 最低谷（新概念密集区）
   │              ● D5: 56  ← 最低谷
   │                      ● D6: 73  ← 触底回升
   │                          ● D7: 63  ← 周总结综合
   └──┬────┬────┬────┬────┬────┬────┬──
     D1   D2   D3   D4   D5   D6   D7

平均：71.7 分  评级：B+
```

**趋势解读**：D4-D5 的 59/56 分是 Week 4 最陡学习曲线的反映——Advisor、Swagger 类名冲突、可观测三支柱这些概念完全是从零开始。D6 整合后回升到 73，说明"分散学习 → 集中整合"的策略有效。

## 本周穿越的 6 个盲区

| # | 盲区 | 来源 | 状态 |
|---|------|------|------|
| 1 | ApiResponse 类名冲突（Java 无 import alias） | D4 Q3 | ✅ 完全限定名 + 踩坑笔记 |
| 2 | 标准排障路径 | D4 Q5 | ✅ D6 Q5 拿到 18/20 |
| 3 | 自定义 Advisor 实现 | D5 Q4 | ✅ 写了 MetricsAdvisor + 7 个测试 |
| 4 | 可观测三支柱（Logs/Metrics/Traces） | D5 Q5 | ✅ 面试级完整回答 |
| 5 | Advisor 是 per-request 不是全局切面 | D6 Q1 | ⚠️ 需持续警惕 |
| 6 | CallAdvisor 方法签名 | D6 Q2 | ⚠️ `adviseCall(ChatClientRequest, CallAdvisorChain)` |

## 本周核心认知（面试可讲）

### 1. BeanOutputConverter 的底层原理
> "Spring AI 扫描 @JsonProperty 注解 → 生成 JSON Schema → 拼入 Prompt → LLM 返回 JSON → Jackson 反序列化为 Java Bean。`.entity(Class)` 一行完成这四步，比 `.content()` + 手动 JSON.parse 更安全——因为 Schema 约束让 LLM 更不容易跑偏。"

### 2. 五层防跑偏策略
> "第一层 @JsonProperty(required=true) 强制字段存在；第二层 @JsonPropertyDescription 让 LLM 理解字段语义；第三层 Few-shot 范例让 LLM 模式匹配；第四层 System 角色约束；第五层后校验（validateAndFallback）。前四层降低跑偏概率，第五层兜底保证下游消费到的数据永远干净。"

### 3. 成本分界线
> "文本统计（字符数/句子数/段落数/阅读时间）用正则 + String 操作，零 Token 成本、100% 准确。标题/关键词/分类/摘要/关键句提取用 AI，因为涉及语义理解。'能程序算的绝不调 AI'——这是成本管控的第一原则。"

### 4. Advisor 拦截链
> "Advisor 是 ChatClient 的中间件，基于责任链模式。SimpleLoggerAdvisor 在 DEBUG 日志记录每次调用的 request/response，自定义 MetricsAdvisor 采集耗时（Timer）+ Token 用量（Counter）到 Micrometer。挂载是 per-request 的，通过 `.advisors(a -> a.advisors(...))` 只在当次调用生效。"

### 5. 管线设计哲学
> "Unix 管道思想——每一段独立可替换。Tika 解析可以换 PDFBox、元数据提取可以换 extractV3、关键句提炼可以换本地模型——改任何一段不影响其他段。"

## 最终项目进度

```
Phase 1 (W1-3): Java/Spring 地基 + 第一次接 LLM ✅
Phase 2 (W4-6): 结构化输出 + RAG
  ├── Week 4: 文档解析 + 结构化抽取 ✅  ← 你在这里
  ├── Week 5: RAG v1 知识库问答 ⬜
  └── Week 6: RAG v2 检索质量工程化 ⬜
Phase 3 (W7-9): Agent + 工作流 + MCP
Phase 4 (W10-12): 工程化 + 求职冲刺
```

**下一步**：Week 5 将 Week 4 的"接入层"产出（结构化文档）存入向量库，跑通 `POST /api/qa/ask`——用户自然语言提问，系统返回带原文引用的答案。

## 关联

- 上位：[[Week4 索引]]
- 总图：[[总图谱]]
- 上一周：[[Week3 索引]]
- 下一周：Week5 索引（待建）
- 核心笔记链：
  - [[BeanOutputConverter 结构化输出]] → [[Prompt 工程进阶（Few-shot + Schema 约束）]] → [[Apache Tika 文档解析]] → [[结构化抽取完整链路]] → [[文档智能分析综合实战]]
  - 可观测链：[[Spring AI Advisors 拦截链]] → [[MetricsAdvisor 自定义指标采集]]
  - 踩坑：[[ApiResponse 类名冲突（Java import 限制）]]
