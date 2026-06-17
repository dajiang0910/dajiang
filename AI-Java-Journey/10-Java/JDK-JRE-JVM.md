# JDK-JRE-JVM

> JDK 包含 JRE,JRE 包含 JVM;三者是层层包含的开发/运行体系。

## 它解决什么问题
搞清楚"我装的到底是什么、运行 Java 需要哪一层",是理解 Java"先编译再运行"模型的起点。

## 核心要点
- **包含关系**:`JDK ⊃ JRE ⊃ JVM`
- **JVM**(Java Virtual Machine):真正执行字节码 `.class` 的虚拟机,屏蔽操作系统差异。
- **JRE**(Java Runtime Environment)= JVM + 核心类库,只能"运行"Java 程序。
- **JDK**(Java Development Kit)= JRE + 开发工具(`javac` 编译器、`jar`、调试器等),能"开发 + 运行"。
- 一句话:**开发用 JDK,只跑程序用 JRE,真正干活的是 JVM。**

## Java 里怎么落地
- 装的是 JDK 21,所以你既能 `javac` 编译又能 `java` 运行。
- `java -version` 看的是 JDK/JRE 版本;字节码最终交给 JVM 执行。

## 面试怎么问
- "JDK、JRE、JVM 区别?" → 答包含关系 + 各自职责。
- 追问:"为什么有了 JVM 还能跨平台?" → 见 [[Java vs 旧语言]] 里的"一次编译到处运行"。

## 关联
- 对照:[[Java vs 旧语言]](字节码 + JIT + 跨平台机制)
- 应用:[[Maven 工程结构]](工程怎么被编译运行)
