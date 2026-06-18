# Maven 工程结构

> Maven 是 Java 的依赖管理 + 构建工具,用 `pom.xml` 描述工程;约定了标准的目录结构。

## 它解决什么问题
统一管理第三方依赖(自动下载)、统一编译/测试/打包流程,让任何人 clone 工程后 `mvn` 一下就能构建。类比:npm / pip + 构建工具。

## 核心要点
- **坐标三元组**(唯一标识一个工程/依赖):
  - `groupId`:组织/团队(如 `com.journey`)
  - `artifactId`:模块名(如 `java-ai-journey`)
  - `version`:版本(如 `1.0.0`)
- **标准目录**:
  - `src/main/java`:正式业务代码(打包上线)
  - `src/test/java`:单元测试代码(只测试用,不打包)
  - `pom.xml`:工程描述(依赖、插件、编译配置)
- **常用生命周期**:`compile`(编译)→ `test`(跑测试)→ `package`(打包成 jar)。
- **国内提速**:在 `~/.m2/settings.xml` 配阿里云镜像,避免拉依赖卡死。

## Java 里怎么落地
Day1 的 `pom.xml` 设 `maven.compiler.release=21`、加 `exec-maven-plugin` 以便命令行跑 main。

## 面试怎么问
- "Maven 坐标三要素?" → groupId / artifactId / version。
- "main 和 test 目录区别?" → test 不参与打包。

## 关联
- 上位:[[JDK-JRE-JVM]]
- 踩坑:[[踩坑-PowerShell 吞 -D 参数]]
- 测试:[[JUnit5]](`mvn test` 跑的测试框架)
