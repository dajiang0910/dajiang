# 分页查询（JPA vs MyBatis-Plus）

> 分页是后端最常见的需求。JPA 用 `Pageable` + `Page<T>`，MyBatis-Plus 用 `Page<T>` + `selectPage`。两者页码起点不同，混用会出 bug。

## JPA 分页

```java
// Repository（不用额外写方法，JpaRepository 自带）
public interface NoteRepository extends JpaRepository<Note, Long> {
    Page<Note> findAll(Pageable pageable);
}

// Service
public Page<Note> list(int page, int size) {
    return noteRepository.findAll(PageRequest.of(page, size));
}

// Controller
@GetMapping("/jpa/page")
public Page<Note> jpaPage(@RequestParam(defaultValue = "1") int page,
                          @RequestParam(defaultValue = "10") int size) {
    return noteService.list(page - 1, size);  // 前端传 1，JPA 需要 -1
}
```

### Page`<`T`>`返回字段

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| `content` | `List<T>` | 当前页数据 |
| `totalElements` | `long` | 总记录数 |
| `totalPages` | `int` | 总页数 |
| `number` | `int` | 当前页码（从 0 开始） |
| `size` | `int` | 每页大小 |
| `numberOfElements` | `int` | 当前页实际条数 |
| `first` | `boolean` | 是否第一页 |
| `last` | `boolean` | 是否最后一页 |
| `empty` | `boolean` | 是否空页 |

### JPA 分页要点

- **页码从 0 开始**：`PageRequest.of(0, 10)` 是第一页
- **返回类型**：`org.springframework.data.domain.Page<T>`
- **自动 COUNT**：JPA 自动生成 `SELECT COUNT(*)` 查询总数

## MyBatis-Plus 分页

```java
// Mapper（继承 BaseMapper 即可）
public interface NoteMapper extends BaseMapper<Note> {}

// Controller（直接调用 selectPage）
@GetMapping("/mp/page")
public IPage<Note> mpPage(@RequestParam(defaultValue = "1") int page,
                          @RequestParam(defaultValue = "10") int size) {
    Page<Note> pageParam = new Page<>(page, size);  // MP 页码从 1 开始
    return noteMapper.selectPage(pageParam, null);
}
```

### IPage`<`T`>` 返回字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `records` | `List<T>` | 当前页数据 |
| `total` | `long` | 总记录数 |
| `pages` | `long` | 总页数 |
| `current` | `long` | 当前页码（从 1 开始） |
| `size` | `long` | 每页大小 |

### MP 分页要点

- **页码从 1 开始**：`new Page<>(1, 10)` 是第一页
- **返回类型**：`com.baomidou.mybatisplus.core.metadata.IPage<T>`
- **必须配置分页插件**：`PaginationInnerInterceptor` 不注册则分页不生效
- **3.5.9 踩坑**：`PaginationInnerInterceptor` 从 extension 移到了 `mybatis-plus-jsqlparser` 模块

## 对比速查

| 维度 | JPA | MyBatis-Plus |
|---|---|---|
| 页码起点 | **0** | **1** |
| 参数类 | `PageRequest.of(page, size)` | `new Page<>(page, size)` |
| 返回类 | `Page<T>` | `IPage<T>` |
| 数据字段 | `content` | `records` |
| 总数字段 | `totalElements` | `total` |
| 总页数 | `totalPages` | `pages` |
| 当前页码 | `number` | `current` |
| 自动 COUNT | ✅ 自动 | ✅ 需注册插件 |

## 分页插件配置（MyBatis-Plus 必须）

```java
@Configuration
@MapperScan("com.example.notes_api.mapper")
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.H2));
        return interceptor;
    }
}
```

不配这个，`selectPage` 会返回全部数据，`total` 为 0。

## JPA + MP 共存时的分页

两个框架的 `Page` 类名相同但包不同，注意区分：

```java
import org.springframework.data.domain.Page;         // JPA 的 Page
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;  // MP 的 Page
```

**建议**：Controller 方法用全限定名或 `IPage` 接口避免冲突。

## 面试怎么问

- "JPA 和 MP 的分页页码有什么区别？" → JPA 从 0 开始，MP 从 1 开始
- "MP 分页不生效怎么回事？" → 没注册 `PaginationInnerInterceptor` 插件
- "分页返回的字段有哪些？" → JPA: content/totalElements/totalPages/number/size；MP: records/total/pages/current/size

## 关联

- JPA：[[Spring Data JPA 与 JpaRepository]]
- MP：[[MyBatis-Plus 核心]]
- 踩坑：MP 3.5.9 `PaginationInnerInterceptor` 移到 `mybatis-plus-jsqlparser`
