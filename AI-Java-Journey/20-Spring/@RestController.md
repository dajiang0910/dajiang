# @RestController

> `@RestController` = `@Controller` + `@ResponseBody`:标注一个类是 Web 入口,且方法返回值**直接序列化成 JSON 写进 HTTP 响应体**(而非当视图名)。

## 它解决什么问题
写 REST API 时,我们要的是「方法 `return` 一个对象 → 自动变 JSON 返回前端」。`@ResponseBody` 就是干这个的;`@RestController` 把它内置了,省得每个方法都标。

## 核心要点
- **拆解**:`@RestController` = `@Controller`(我是 Web 组件,也是个 Bean)+ `@ResponseBody`(返回值进响应体)。
- **@ResponseBody 的作用**:返回值交给 `HttpMessageConverter`(底层 Jackson)序列化成 JSON,直接写响应体。
- **没有 @ResponseBody 会怎样(Day1 Q2 纠偏)**:纯 `@Controller` 时,返回值被当成**视图名(view name)**交给 `ViewResolver` 找页面模板 → 找不到模板 → Whitelabel Error Page 报错。**所以现象「报错」对,但原因是「返回值被当视图名解析」,不是「方法没有返回值」。**
- **常用配套注解**:
  - `@RequestMapping("/api/notes")`:类级公共前缀,方法上的路径在它后面拼。
  - `@GetMapping("/{id}")`:映射 GET 请求;`{id}` 是路径占位符。
  - `@PostMapping`/`@PutMapping`/`@DeleteMapping`:映射 POST/PUT/DELETE。
  - `@PathVariable Long id`:把 URL 里的 `{id}` 抽出来、自动转类型。
  - `@RequestParam`:查询参数(`?name=xxx`)。
  - `@RequestBody`:请求体 JSON 自动反序列化成 Java 对象(详见下方)。
  - `@Valid`:触发 [[Bean Validation]] 校验,校验失败抛 `MethodArgumentNotValidException`。

## @RequestBody 详解

把 HTTP 请求体(JSON)映射到 Java 对象:
```java
@PostMapping
public ApiResponse<NoteResponse> create(@Valid @RequestBody CreateNoteRequest request) {
    // request 已经是 Java 对象了,直接用
    Note note = noteService.create(request.title(), request.content());
    return ApiResponse.success(NoteResponse.from(note));
}
```

**工作流程**:
1. 客户端发送 JSON: `{"title": "笔记", "content": "内容"}`
2. Jackson 把 JSON 反序列化为 `CreateNoteRequest` record
3. `@Valid` 触发 [[Bean Validation]] 校验
4. 校验通过才执行方法体,校验失败返回 400 + 错误信息

**`@RequestBody` vs `@RequestParam`**:

|     | `@RequestBody` | `@RequestParam`        |
| --- | -------------- | ---------------------- |
| 来源  | HTTP 请求体(JSON) | URL 查询参数(`?key=value`) |
| 格式  | JSON → Java 对象 | 字符串 → 单个参数             |
| 场景  | POST/PUT 请求    | GET 请求                 |

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
- 进阶：[[Spring AI 起步]] —— ChatController 同样遵循三层架构，注入 ChatService 处理 LLM 对话
