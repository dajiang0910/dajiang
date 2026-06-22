# @SpringBootApplication

> Spring Boot 启动类上的「三合一」注解。Day1 先知道它是入口标志 + 三注解轮廓,自动配置原理留到 Day6 深挖。

## 核心要点(先建立轮廓)
`@SpringBootApplication` = 三个注解的组合:
- `@SpringBootConfiguration`:声明这是个配置类(本质是 `@Configuration`)。
- `@EnableAutoConfiguration`:**自动配置**的开关 —— 根据 classpath 上的依赖(如 `spring-web`)自动装配 Tomcat、Jackson 等。这是 Spring Boot「开箱即用」的核心魔法。
- `@ComponentScan`:从**启动类所在包**开始,向下扫描 `@Component/@Service/@RestController`,注册成 Bean。

> ⚠️ 因为 `@ComponentScan` 从启动类所在包往下扫,**你的 Controller/Service 必须放在启动类的同级或子包下**,否则扫不到、注入失败。这是新手常踩的「Bean 找不到」坑。

## Java 里怎么落地
```java
@SpringBootApplication            // 放在根包 com.example.notesapi 下
public class NotesApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(NotesApiApplication.class, args);
    }
}
```

## 面试怎么问
- "`@SpringBootApplication` 包含哪三个注解?" → `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan`。
- "自动配置原理?" → Day6 / W2 后期细讲(`AutoConfiguration.imports` 清单 + `@Conditional` 条件装配)。

## 关联
- 工程入口:[[Spring Boot 3 起手]]
- 容器机制:[[Spring IoC 容器]]
- TODO(Day6 深挖):自动配置原理 + 条件装配
