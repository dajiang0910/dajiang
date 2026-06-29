# MockMvc 控制器测试

> Spring 提供的**模拟 HTTP 请求**工具，用于测试 Controller 层，不需要启动真正的服务器。

## 它解决什么问题

测 Controller 时，不想启动 Tomcat、不想连数据库。MockMvc 模拟一个浏览器发送请求：

```java
mockMvc.perform(get("/api/notes"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$[0].title").value("测试标题"));
```

等价于用 curl 访问 `GET /api/notes`，但在测试里自动完成。

## 三种测试 Controller 的方式

| 方式 | 加载容器 | 启动速度 | 适合场景 |
|---|---|---|---|
| `@WebMvcTest` + MockMvc | 轻量（只 Controller） | 快 | **单元测试（推荐）** |
| `@SpringBootTest` + MockMvc | 完整 | 慢 | 集成测试 |
| `@SpringBootTest` + `TestRestTemplate` | 完整 + 真实服务器 | 最慢 | 端到端测试 |

## @WebMvcTest 核心机制

```java
@WebMvcTest(NoteController.class)   // 只加载这一个 Controller
class NoteControllerTest {

    @Autowired
    MockMvc mockMvc;                 // 自动注入，模拟 HTTP

    @MockitoBean
    NoteService noteService;         // 假的 Service（不加载真正的 Bean）

    @MockitoBean
    NoteMapper noteMapper;           // 假的 Mapper
}
```

`@WebMvcTest` 做的事：
1. 创建轻量级 Spring 容器
2. 只注册 `NoteController` 相关的 Bean
3. 自动配置 `MockMvc`
4. **不加载** Service、Repository、数据库等

所以 Service 和 Mapper 必须用 `@MockitoBean` 手动提供假的。

## Spring Boot 4.x 注解变化

| 注解 | 3.x 包路径 | 4.x 包路径 |
|---|---|---|
| `@WebMvcTest` | `o.s.b.test.autoconfigure.web.servlet` | `o.s.b.webmvc.test.autoconfigure` |
| `@MockBean`（已废弃） | `o.s.b.test.mock.bean` | → `@MockitoBean` `o.s.t.context.bean.override.mockito` |

## MockMvc 常用 API

```java
// GET 请求
mockMvc.perform(get("/api/notes"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$[0].title").value("标题"));

// POST 请求（带 JSON body）
mockMvc.perform(post("/api/notes")
       .contentType(MediaType.APPLICATION_JSON)
       .content("""
           {"title": "新标题", "content": "新内容"}
           """))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.code").value(200));

// 路径参数
mockMvc.perform(get("/api/notes/1"))
       .andExpect(jsonPath("$.data.title").value("标题"));

// 参数校验失败
mockMvc.perform(post("/api/notes")
       .contentType(MediaType.APPLICATION_JSON)
       .content("""
           {"title": "", "content": "内容"}
           """))
       .andExpect(status().isBadRequest());
```

## jsonPath 表达式

| 表达式 | 含义 |
|---|---|
| `$.code` | 顶层字段 |
| `$.data.title` | 嵌套字段 |
| `$[0].title` | 数组第一个元素的字段 |
| `$.size()` | 数组长度 |

## 实战代码（Day 6）

```java
@WebMvcTest(NoteController.class)
class NoteControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    NoteService noteService;

    @MockitoBean
    NoteMapper noteMapper;

    @Test
    @DisplayName("POST /api/notes 标题为空应返回 400")
    void create_withBlankTitle_shouldReturn400() throws Exception {
        mockMvc.perform(post("/api/notes")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"title": "", "content": "内容"}
                                """))
                .andExpect(status().isBadRequest());
    }
}
```

## 关联
- 前置:[[Mockito 单元测试]]（@MockitoBean 替代了 @Mock 的角色）
- 前置:[[@RestController]]（被测对象）
- 对比:[[@SpringBootTest]]（全量加载，集成测试用）
- 代码: `notes-api/src/test/java/.../controller/NoteControllerTest.java`
- 测哪一层：[[三层架构(Controller-Service-Repository)]] 的 Controller 层
- DI 原理：[[依赖注入(DI)]] 的构造器注入 → `@MockitoBean` 替换容器中的 Bean
- 校验测试：[[Bean Validation]] 的 `@NotBlank` → 用 JSON body 触发校验断言 400
- Day 6 补齐：[[综合实战（智能笔记助手）]] —— 新增 `ChatControllerTest.java`，12 个测试覆盖全部 8 个 chat 端点，`@WebMvcTest` + `@MockitoBean` 模拟 ChatService，不依赖真实 API Key
