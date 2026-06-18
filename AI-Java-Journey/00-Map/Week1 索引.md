# Week 1 索引

> **Java 速通 + 工程环境**。本周目标:跨过 Java 语法关,建立 Maven 工程直觉,产出一个带单测的 CLI 小工具。

## Day 1:Java 工程环境 + 第一个程序 ✅
- [[JDK-JRE-JVM]] —— 开发/运行体系的层层包含关系
- [[Java vs 旧语言]] —— 字节码 + JIT + 跨平台(一次编译到处运行)
- [[Maven 工程结构]] —— 坐标三元组 + 标准目录
- [[record]] —— 不可变数据类,自动生成样板方法
- [[var 与静态类型]] —— 类型推断,类型固定但值可变
- [[踩坑-PowerShell 吞 -D 参数]] —— mvn -D 参数加双引号

## Day 2:类型系统与集合 ✅
- [[泛型]] —— 编译期类型检查 + 免强转
- [[Java 集合框架]] —— List / Set / Map 三大容器
- [[Optional]] —— 用类型表达"可能没有",防 NPE
- [[equals 与 hashCode]] —— 集合去重的底层机制(面试核心)

## Day 3:Stream + Lambda ✅
- [[Stream API]] —— 声明式处理集合(filter/map/collect)
- [[惰性求值]] —— 中间操作是图纸,终端操作才启动(面试核心)
- [[Lambda 与方法引用]] —— \`Doc::title\` 是 \`d -> d.title()\` 的语法糖
- [[Collectors]] —— toList / groupingBy / counting / joining

## Day 4:异常 + try-with-resources + 日志 ✅
- [[Java 异常体系]] —— checked vs unchecked / Error vs Exception
- [[try-with-resources]] —— AutoCloseable 自动关流
- [[SLF4J 日志]] —— 5 个级别,生产 INFO,改配置切级别/目标

## Day 5:HttpClient + Jackson ✅
- [[HttpClient]] —— send() 不查状态码,要自己处理
- [[Jackson]] —— readTree(树) vs readValue(转对象)
- [[踩坑-HttpResponse 不能 try-with-resources]] —— javap 反编译求证,JDK 21 不可用

## Day 6(周末):整合 + 单测 ✅
- [[JUnit5]] —— @Test + 三大断言,mvn test 全绿
- [[picocli]] —— 注解驱动的 CLI 解析,--help / 必填 / 退出码
- [[WeatherService 设计模式]] —— I/O 与业务分离,Service 单测可达

## Day 7(周末):重构 + 复盘 ✅
- 21 知识点双链巡检:9 处断链全补(详见 [[record]] / [[Java 异常体系]] / [[HttpClient]] / [[Jackson]] / [[JUnit5]] / [[WeatherService 设计模式]] 等)
- 新建 [[00-Map/总图谱]] 雏形:12 周转型地图
- 新建 [[踩坑-HttpClient 状态码漏判]] —— send() 不会自动检查状态码,必须自己判
- 5 道自测平均 68%:概念到位、工程深度还要补
- 暴露真问题:try-with-resources 不熟 → Week 2 Day 1 补课

## Week 1 复盘(100-200 字)

**最大收获**:
- Java 数据载体默认 record(自动 equals/hashCode,省掉"只重写一个"坑)
- Stream + 惰性求值是后续 RAG/Agent 流式思维基础
- **I/O 与业务分离**是 Week 2 Spring 三层架构的前置

**踩过最深的坑**:
- HttpClient.send() **不查状态码**(Week 3 接 LLM 必再踩)
- Windows 默认 GBK 编码,中文全乱码(pom.xml 加 UTF-8 解决)
- picocli 没打进 jar(`NoClassDefFoundError`,要 shade 插件)

**没掌握的**:
- try-with-resources 对"非 I/O 资源"不熟
- record vs Lombok 区别 8 股深度不够
- 状态码漏判的回归测试难写(必学 Mockito)

**Week 2 重点**:`@SpringBootApplication` + 三层架构(Controller/Service/Mapper) + 5 个 8 股点(Bean 生命周期、循环依赖、事务传播、异常体系、配置加载)。

## 本周里程碑
- [x] CLI 工具:读配置 → 调 API → Jackson 解析 → 打印结构化结果
- [x] `mvn test` 全绿
- [x] `java -jar target/java-ai-journey-1.0.jar --city 上海` 一行跑通

## 导航
- 上位:[[00-Map/总图谱]]
- 下一周:[[Week2 索引]](待建)
