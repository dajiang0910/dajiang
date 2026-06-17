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

## Day 6(周末):整合 + 单测(待学)
- 预计:[[JUnit5]]

## Day 7(周末):重构 + 复盘
- 补全双链,巩固知识网络

## 本周里程碑
- [ ] CLI 工具:读配置 → 调 API → Jackson 解析 → 打印结构化结果
- [ ] \`mvn test\` 全绿

## 导航
- 上位:[[00-Map/总图谱]](待建)
- 下一周:[[Week2 索引]](待建)
