# 统一响应体（ApiResponse）

> **统一响应体**是前后端约定的返回格式，用泛型 `ApiResponse<T>` 包裹所有接口响应，让前端有统一的拦截逻辑。

## 为什么需要统一响应体

没有统一响应体时，前端需要判断两种格式：
```json
// 成功：直接返回数据
{"id": 1, "title": "笔记"}

// 失败：Spring 默认错误格式
{"timestamp": "...", "status": 404, "error": "Not Found", "path": "/api/notes/99"}
```

有了统一响应体，所有接口返回同一个格式：
```json
// 成功
{"code": 200, "message": "success", "data": {"id": 1, "title": "笔记"}}

// 失败
{"code": 404, "message": "Note not found: 99"}
```

## 定义 ApiResponse

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  // data 为 null 时不序列化
public record ApiResponse<T>(
    int code,
    String message,
    T data
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "success", data);
    }

    public static <T> ApiResponse<T> error(int code, String message) {
        return new ApiResponse<>(code, message, null);
    }
}
```

**关键点**：
- `@JsonInclude(NON_NULL)` —— `data` 为 null 时不输出，保持 JSON 干净
- 泛型 `T` —— 支持任意数据类型
- 静态工厂方法 —— `ApiResponse.success(data)` / `ApiResponse.error(404, msg)`

## 在 Controller 中使用

```java
// 返回单条（带 ApiResponse 包装）
@GetMapping("/{id}")
public ApiResponse<NoteResponse> getById(@PathVariable Long id) {
    Note note = noteService.getById(id);
    return ApiResponse.success(NoteResponse.from(note));
}

// 返回列表（不包装，直接返回数组）
@GetMapping
public List<NoteResponse> list() {
    return noteService.list().stream()
            .map(NoteResponse::from)
            .toList();
}
```

**注意**：列表接口一般不包装，直接返回数组。单条/创建/更新接口用 `ApiResponse` 包裹。

## 前端收益

```javascript
// 统一拦截器
axios.interceptors.response.use(
    response => {
        const { code, message, data } = response.data;
        if (code !== 200) {
            showToast(message);  // 统一错误提示
            return Promise.reject(message);
        }
        return data;  // 只返回业务数据
    },
    error => {
        showToast("网络错误");
        return Promise.reject(error);
    }
);
```

## @JsonInclude 常用值

| 值 | 含义 |
|---|---|
| `NON_NULL` | 值为 null 时不序列化 |
| `NON_EMPTY` | 值为 null/空字符串/空集合时不序列化 |
| `NON_DEFAULT` | 值为默认值时不序列化（int 默认 0，boolean 默认 false） |

## 面试怎么问

- "为什么需要统一响应体？" → 前端统一拦截、统一错误处理、减少联调成本
- "ApiResponse 怎么设计？" → code + message + data，用泛型支持任意类型
- "为什么用 `@JsonInclude(NON_NULL)`？" → error 时不输出 data 字段，保持 JSON 干净

## 关联

- 搭配使用：[[Bean Validation]]、[[全局异常处理]]
- 被包装：[[DTO 与 Entity 之分]] 的 NoteResponse
- 返回位置：[[@RestController]] 的方法
- 测试验证：[[MockMvc 控制器测试]] 中用 `jsonPath("$.code").value(200)` 断言响应格式
- 新端点复用：[[Spring AI 起步]] —— `POST /api/chat` 同样返回 `ApiResponse<String>`，一套响应体覆盖全部 API
