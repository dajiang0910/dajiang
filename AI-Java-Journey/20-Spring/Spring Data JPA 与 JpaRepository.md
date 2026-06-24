# Spring Data JPA 与 JpaRepository

> Spring Data JPA 是 Spring 对 JPA（Java Persistence API）的封装。继承 `JpaRepository` 接口，即拥有完整 CRUD，**0 行实现代码**。

## 它解决什么问题

手写 `NoteRepository` 实现类需要 30+ 行（`findAll`、`findById`、`save`、`deleteById`……），Spring Data JPA 把这些全自动化了：

```java
// 之前（Day 2-3）：手写 30+ 行实现
@Component
public class InMemoryNoteRepository implements NoteRepository {
    private final Map<Long, Note> store = new HashMap<>();
    public List<Note> findAll() { return new ArrayList<>(store.values()); }
    public Optional<Note> findById(Long id) { return Optional.ofNullable(store.get(id)); }
    public Note save(Note note) { ... }
    // ...
}

// 现在（Day 4）：继承即拥有，0 行实现
public interface NoteRepository extends JpaRepository<Note, Long> {
    // JpaRepository 已经提供了 findAll, findById, save, deleteById, count, existsById ...
}
```

## 泛型参数含义

```java
JpaRepository<Note, Long>
//             ↑      ↑
//        实体类    主键类型
```

- **Note**：要操作的数据库实体（对应 `@Entity` 注解的类）
- **Long**：实体主键 `@Id` 字段的数据类型

## JpaRepository 提供的方法

| 方法 | 作用 | SQL 等价 |
|---|---|---|
| `findAll()` | 查询所有 | `SELECT * FROM notes` |
| `findById(id)` | 按 ID 查询（返回 `Optional`） | `SELECT * FROM notes WHERE id = ?` |
| `save(entity)` | 新增或更新 | INSERT 或 UPDATE |
| `deleteById(id)` | 按 ID 删除 | `DELETE FROM notes WHERE id = ?` |
| `count()` | 总数 | `SELECT COUNT(*) FROM notes` |
| `existsById(id)` | 判断是否存在 | `SELECT 1 FROM notes WHERE id = ?` |

## save() 的 INSERT vs UPDATE

```java
noteRepository.save(new Note("标题", "内容"));     // id == null → INSERT
noteRepository.save(existingNote);                  // id != null → UPDATE
```

判断依据：Hibernate 检查 `@Id` 字段是否为 null。
- `null` → 数据库自动生成 ID（IDENTITY 策略）→ INSERT
- 非 null → SELECT 检查是否存在 → 存在则 UPDATE

## Service 层零修改

```java
// NoteService 一个字都不用改——因为依赖的是接口，不是实现
@Service
public class NoteService {
    private final NoteRepository noteRepository;  // 接口类型
    public NoteService(NoteRepository noteRepository) {
        this.noteRepository = noteRepository;
    }
    // findAll / findById / save / deleteById 签名没变 → Service 不用改
}
```

这就是 [[面向接口编程]] 的实际价值——换数据库实现时 Service 一行不用改。

## 自定义查询方法

除了继承的方法，还可以按命名约定自定义查询：

```java
public interface NoteRepository extends JpaRepository<Note, Long> {
    // Spring Data 自动根据方法名生成 SQL
    List<Note> findByTitle(String title);                    // WHERE title = ?
    List<Note> findByTitleContaining(String keyword);        // WHERE title LIKE %keyword%
    Optional<Note> findByTitleAndContent(String t, String c); // WHERE title = ? AND content = ?
}
```

方法名约定：`findBy` + 字段名（驼峰），Spring Data 自动解析为 SQL。

## Day 4 实战：从内存到数据库

Day 4 的改动：

1. `pom.xml` 加 `spring-boot-starter-data-jpa` + `h2` 依赖
2. `Note.java` 加 `@Entity` / `@Table` / `@Id` / `@GeneratedValue` / `@Column` 注解
3. `NoteRepository` 从 30 行实现类 → 1 行接口（`extends JpaRepository`）
4. 删除 `InMemoryNoteRepository.java`
5. `NoteService` **零修改**

## 面试怎么问

- "JpaRepository 提供了哪些方法？" → findAll, findById, save, deleteById, count, existsById
- "save() 什么时候 INSERT 什么时候 UPDATE？" → id 为 null INSERT，非 null UPDATE
- "Spring Data JPA 怎么自定义查询？" → 方法名约定（findBy + 字段名）

## 关联

- 底层依赖：[[JPA 实体注解]]
- 数据库：[[H2 内存数据库]]
- 配置：[[application.properties 配置]]
- 思想：[[面向接口编程]]
- 对比：[[MyBatis-Plus]]（Day 5 学习）
