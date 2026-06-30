# ApiResponse 类名冲突（Java import 限制）

> 项目 DTO 取名 `ApiResponse`，Swagger 注解也叫 `@ApiResponse`，编译器报歧义。Java 没有 `import...as` 别名机制，解决方案：完全限定名 or DTO 改名。

## 症状

```java
// 项目中定义了统一响应体
package com.example.notes_api.dto;
public record ApiResponse<T>(int code, String message, T data) {}

// Controller 中想用 Swagger 注解
import io.swagger.v3.oas.annotations.responses.ApiResponse;  // ← 和上面的同名！

@ApiResponse(responseCode = "200")  // ← 编译器报：reference to ApiResponse is ambiguous
public ApiResponse<NoteMetadata> parseAndExtract(...) {}
```

编译器错误：
```
error: reference to ApiResponse is ambiguous
  both class com.example.notes_api.dto.ApiResponse and
  class io.swagger.v3.oas.annotations.responses.ApiResponse match
```

## 根本原因

Java 的 `import` 只是**命名空间缩写**，不提供别名机制。这是 Java 的刻意设计——"显式优于隐式"。

对比其他语言：
```python
# Python：有 import...as
import numpy as np
from myapp.dto import ApiResponse as MyApiResponse

# TypeScript：有 import...as
import { ApiResponse as SwaggerApiResponse } from 'swagger-annotations';
```

Java 没有等价物。同一个源文件中 import 两个同名类 → 至少有一个必须用**完全限定名**。

## 解决方案

### 方案 1：完全限定名（当前采用）

```java
// 不 import Swagger 的 ApiResponse，用完全限定名
@io.swagger.v3.oas.annotations.responses.ApiResponses({
    @io.swagger.v3.oas.annotations.responses.ApiResponse(
        responseCode = "200",
        description = "解析成功",
        content = @Content(schema = @Schema(implementation = DocumentParseResponse.class))
    ),
    @io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "400", description = "文件为空"),
})
public ApiResponse<DocumentParseResponse> parse(...) {}
```

**优点**：不改 DTO 名，零风险
**缺点**：注解冗长，可读性下降

### 方案 2：DTO 改名（推荐长期方案）

```java
// 把 ApiResponse 改成项目中不会冲突的名字
// 选项 A：CommonResponse（通用响应）
public record CommonResponse<T>(int code, String message, T data) {}

// 选项 B：R（短名，类似 AjaxResult）
public record R<T>(int code, String message, T data) {}

// 选项 C：Result
public record Result<T>(int code, String message, T data) {}
```

**优点**：一劳永逸，代码清爽
**缺点**：改 DTO 名需要全局替换（影响所有 Controller、测试、ExceptionHandler）

### 方案 3：拆分模块（大项目适用）

```java
// 把 Swagger 配置集中在单独的配置类/接口中
// Controller 只引用自定义注解，避免 import 冲突
```

## 经验教训

1. **命名时检查第三方库的常用类型**——给 DTO 取名叫 `ApiResponse` 之前，先 grep 一下 Swagger/Spring 的 jar 里有没有同名类
2. **统一响应体建议用 `R` 或 `Result`**——国内很多 Spring Boot 项目用 `R`，短且不冲突
3. **Java 没有 `import...as`** 是设计选择，不是缺陷——它迫使你正视命名冲突而不是用别名掩盖

## 关联

- 上游：[[统一响应体（ApiResponse）]]（Week 2 创建的 DTO）
- 相关：[[Swagger 文件上传与文档测试]]（Week 4 Day 4 首次遇到此冲突）
- 位置：[[结构化抽取完整链路]]（Day 4 工程化时暴露）
- 底层：[[Java vs 旧语言]]（Java 显式 > 隐式的设计哲学）
