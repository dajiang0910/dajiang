# Mockito 单元测试

> Java 主流 Mock 框架，用于**隔离被测代码**，用假对象替代真实依赖（数据库、网络、其他 Service）。

## 它解决什么问题

测 `NoteService.create()` 时，不想真的连数据库。Mockito 创建一个假的 `NoteRepository`，让它按剧本返回数据：

```java
// 告诉 Mockito：当 save() 被调用时，返回这个假对象
when(mockRepository.save(any(Note.class))).thenReturn(savedNote);
```

这样测试的是 **Service 自身逻辑**，不依赖任何外部因素。

## 测试金字塔

```
        ╱  E2E  ╲         ← 少量，慢，测全链路
       ╱─────────╲
      ╱  集成测试  ╲       ← 适量，中速，测层间协作
     ╱─────────────╲
    ╱   单元测试     ╲     ← 大量，快，测单个类
   ╱─────────────────╲
```

单元测试是最底层、最重要、跑得最快的。

## 核心注解

| 注解 | 作用 | 类比 |
|---|---|---|
| `@ExtendWith(MockitoExtension.class)` | 启用 Mockito | 开关 |
| `@Mock` | 创建假对象（替身演员） | 替身 |
| `@InjectMocks` | 创建真实对象，自动注入 `@Mock` 依赖 | 真人 + 替身搭戏 |
| `when(...).thenReturn(...)` | 设置假对象的行为 | 写剧本 |
| `verify(mock, times(1))` | 验证方法是否被调用 | 监控回放 |

## @InjectMocks 自动注入机制

```java
@Mock
NoteRepository noteRepository;   // 假的仓库

@Mock
NoteMapper noteMapper;           // 假的 Mapper

@InjectMocks
NoteService noteService;         // 真实的 Service
```

`@InjectMocks` 做的事：
1. 创建**真实的** `NoteService` 对象
2. 扫描它的构造函数参数
3. 找到匹配的 `@Mock` 自动注入

等价于：`new NoteService(noteRepository, noteMapper)` —— Mockito 自动匹配。

## 实战代码（Day 6）

```java
@ExtendWith(MockitoExtension.class)
class NoteServiceTest {

    @Mock
    NoteRepository noteRepository;

    @InjectMocks
    NoteService noteService;

    @Test
    @DisplayName("getById 不存在时应抛异常")
    void getById_whenNotExists_shouldThrowException() {
        // given：findById 返回空
        when(noteRepository.findById(999L)).thenReturn(Optional.empty());

        // when & then：调用应该抛异常
        RuntimeException ex = assertThrows(RuntimeException.class,
                () -> noteService.getById(999L));
        assertTrue(ex.getMessage().contains("999"));
    }

    @Test
    @DisplayName("create 应保存并返回新笔记")
    void create_shouldSaveAndReturnNote() {
        // given
        Note savedNote = new Note("新标题", "新内容");
        when(noteRepository.save(any(Note.class))).thenReturn(savedNote);

        // when
        Note result = noteService.create("新标题", "新内容");

        // then
        assertEquals("新标题", result.getTitle());
        verify(noteRepository, times(1)).save(any(Note.class));
    }
}
```

## Mock 常用 API

```java
// 设置返回值
when(mock.method(args)).thenReturn(value);

// 设置抛异常
when(mock.method(args)).thenThrow(new RuntimeException("boom"));

// 匹配任意参数
when(mock.save(any(Note.class))).thenReturn(saved);

// 验证调用次数
verify(mock, times(1)).deleteById(1L);
verify(mock, never()).deleteById(2L);
```

## 关键区别

| | `@ExtendWith(MockitoExtension)` | `@WebMvcTest` | `@SpringBootTest` |
|---|---|---|---|
| 加载 Spring 容器 | ❌ 不加载 | ✅ 轻量（只 Controller） | ✅ 完整 |
| 需要数据库 | ❌ | ❌ | ✅ |
| 启动速度 | 极快 | 快 | 慢 |
| 适合测 | Service 层 | Controller 层 | 全链路 |

## 关联
- 前置:[[JUnit5]]（JUnit 是测试框架，Mockito 是 Mock 框架，配合使用）
- 应用:[[MockMvc 控制器测试]]（Controller 层用 MockMvc + @MockitoBean）
- 代码: `notes-api/src/test/java/.../service/NoteServiceTest.java`
- 测哪一层：[[三层架构(Controller-Service-Repository)]] 的 Service 层
- DI 原理：[[依赖注入(DI)]] 的构造器注入 → `@InjectMocks` 模拟自动注入
- 异常断言：[[全局异常处理]] 中的 `BusinessException` → `assertThrows(BusinessException.class, ...)`
