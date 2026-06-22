# Spring Boot 3 起手

> 用 `start.spring.io` 生成工程,一个 `@SpringBootApplication` + `main` 里 `SpringApplication.run(...)`,就能起一个内嵌 Tomcat 的 Web 服务,开箱即用。

## 它解决什么问题
传统 Spring 要手写一堆 XML 配置、自己装 Tomcat、配 `web.xml`。Spring Boot 用「约定优于配置 + 自动配置 + 内嵌容器」把这些全省了:你只管写业务,框架帮你把脚手架搭好。
- 类比:像 Python 的 FastAPI / Node 的 Express 那样「几行起一个服务」,但背后是企业级 Spring 生态。

## 核心要点
- **生成方式**:https://start.spring.io 选 Maven + Java 21 + Spring Boot 3.x,依赖按需勾(Day1 只勾 Spring Web)。
- **启动类**:带 `@SpringBootApplication` 的类,`main` 里调 `SpringApplication.run(本类.class, args)`。
- **这一行 run() 做了三件事**:① 启动 IoC 容器 ② 扫描注册所有 Bean ③ 启动内嵌 Tomcat 监听 8080。
- **打包**:用 `spring-boot-maven-plugin` 打成「可执行 fat jar」,`java -jar xxx.jar` 直接跑(和 Week1 的 shade 插件目的类似,但这是 Spring 官方插件)。
- **独立工程**:和 Week1 的 CLI 工程**分开建**(`notes-api/`),因为打包插件、依赖体系都不同,硬塞一起会冲突。

## Java 里怎么落地
本项目 `notes-api` 启动类:
```java
@SpringBootApplication
public class NotesApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(NotesApiApplication.class, args);
    }
}
```
验收:`mvn spring-boot:run` → 控制台出现 `Started NotesApiApplication in x seconds` → 浏览器访问 `/api/hello` 返回 JSON。

## 面试怎么问
- "Spring Boot 比传统 Spring 好在哪?" → 自动配置 + 起步依赖(starter)+ 内嵌容器,约定优于配置。
- "`spring-boot-starter-web` 引入了什么?" → Spring MVC + 内嵌 Tomcat + Jackson(JSON 序列化)一整套。

## 关联
- 注解拆解:[[@SpringBootApplication]]
- 容器机制:[[Spring IoC 容器]]
- Web 入口:[[@RestController]]
- 工程基座:[[Maven 工程结构]](Spring Boot 用 `spring-boot-starter-parent` 统一管依赖版本)
