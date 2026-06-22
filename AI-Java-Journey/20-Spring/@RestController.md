# @RestController

> `@RestController` = `@Controller` + `@ResponseBody`:标注一个类是 Web 入口,且方法返回值**直接序列化成 JSON 写进 HTTP 响应体**(而非当视图名)。

## 它解决什么问题
写 REST API 时,我们要的是「方法 `return` 一个对象 → 自动变 JSON 返回前端」。`@ResponseBody` 就是干这个的;`@RestController` 把它内置了,省得每个方法都标。

## 核心要点
- **拆解**:`@RestController` = `@Controller`(我是 Web 组件,也是个 Bean)+ `@ResponseBody`(返回值进响应体)。
- **@ResponseBody 的作用**:返回值交给 `HttpMessageConverter`(底层 Jackson)序列化成 JSON,直接写响应体。
- **没有 @ResponseBody 会怎样(Day1 Q2 纠偏)**:纯 `@Controller` 时,返回值被当成**视图名(view name)**交给 `ViewResolver` 找页面模板 → 找不到模板 → Whitelabel Error Page 报错。**所以现象「报错」对,但原因是「返回值被当视图名解析」,不是「方法没有返回值」。**
- **常用配套注解**:`@GetMapping/@PostMapping`(映射 HTTP 方法+路径)、`@RequestParam`(查询参数)、`@PathVariable`(路径变量)、`@RequestBody`(请求体转对象)。

## Java 里怎么落地
本项目:
```java
@RestController
public class HelloController {
    @GetMapping("/api/hello")
    public Map<String,Object> hello(@RequestParam(defaultValue="同学") String name) {
        return Map.of("message", greetingService.greet(name),
                      "framework", "Spring Boot 3");
    }
}
```
访问 `/api/hello` → 自动返回 `{"message":"...","framework":"Spring Boot 3"}`。

## 踩过的坑
- 把 `@RestController` 误写成 `@Controller` → 接口返回 JSON 变成「找视图」报错。**记牢:REST API 用 `@RestController`,要返回页面才用 `@Controller`。**

## 面试怎么问
- "`@RestController` 和 `@Controller` 区别?" → 前者 = 后者 + `@ResponseBody`,返回值直接当响应体(JSON)。
- "Spring MVC 一个请求怎么走?" → DispatcherServlet 接收 → HandlerMapping 找到 Controller 方法 → 执行 → `@ResponseBody` 走 HttpMessageConverter 序列化 → 写响应。(W2 后期细讲)

## 关联
- 容器机制:[[Spring IoC 容器]]
- 注入:[[依赖注入(DI)]]
- 工程入口:[[Spring Boot 3 起手]]
- 数据载体:[[record]](收发 JSON 常用 record 当 DTO)
