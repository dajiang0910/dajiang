# MyBatis-Plus 核心

> MyBatis-Plus 是 MyBatis 的增强框架，继承 `BaseMapper` 即拥有 CRUD。国内主流 ORM，复杂 SQL 场景比 JPA 更灵活。

## 它和 JPA 的区别

| 维度      | JPA（Spring Data JPA）        | MyBatis-Plus         |
| ------- | --------------------------- | -------------------- |
| 思路      | **面向对象**，操作实体自动生成 SQL       | **面向 SQL**，手写或半自动生成  |
| 基础接口    | `JpaRepository<Entity, Id>` | `BaseMapper<Entity>` |
| 自定义查询   | 方法名约定（`findByTitle`）        | `@Select` 注解或 XML    |
| 复杂 SQL  | `@Query`（JPQL，不直观）          | 原生 SQL（自由度高）         |
| 多表 JOIN | 弱（JPQL 不直观）                 | **强**（直接写 SQL）       |
| 生态      | 国际主流                        | **国内主流**（阿里系）        |
| 适合场景    | 单表 CRUD 为主、DDD              | 复杂查询多、SQL 需调优        |

**面试高频题**："JPA 和 MyBatis-Plus 怎么选？"→ 单表 CRUD 选 JPA，多表 JOIN 选 MP。

## 核心注解

```java
@TableName("notes")           // 映射表名（JPA 用 @Entity + @Table）
public class Note {
    @TableId(type = IdType.AUTO)  // 主键 + 自增（JPA 用 @Id + @GeneratedValue）
    private Long id;

    @TableField("title")          // 字段映射（JPA 用 @Column）
    private String title;
}
```

| MyBatis-Plus | JPA 等价 | 作用 |
|---|---|---|
| `@TableName("表名")` | `@Entity` + `@Table` | 映射表 |
| `@TableId(type=AUTO)` | `@Id` + `@GeneratedValue` | 主键自增 |
| `@TableField("列名")` | `@Column` | 字段映射 |

## BaseMapper 提供的方法

```java
public interface NoteMapper extends BaseMapper<Note> {
    // 0 行代码，BaseMapper 已提供全部基础 CRUD
}
```

| 方法 | 作用 | SQL 等价 |
|---|---|---|
| `selectById(id)` | 按 ID 查询 | `SELECT * FROM notes WHERE id = ?` |
| `insert(entity)` | 新增 | `INSERT INTO notes ...` |
| `updateById(entity)` | 按 ID 更新（null 字段不更新） | `UPDATE notes SET ...` |
| `deleteById(id)` | 按 ID 删除 | `DELETE FROM notes WHERE id = ?` |
| `selectList(wrapper)` | 条件查询 | `SELECT * FROM notes WHERE ...` |
| `selectPage(page, wrapper)` | 分页查询 | `SELECT * FROM notes LIMIT ...` |

## 自定义复杂查询

```java
public interface NoteMapper extends BaseMapper<Note> {

    // 注解方式写原生 SQL
    @Select("SELECT * FROM notes WHERE title LIKE CONCAT('%', #{keyword}, '%')")
    List<Note> searchByTitle(@Param("keyword") String keyword);

    // 多表 JOIN（JPA 做这个很痛苦）
    @Select("SELECT n.*, t.name AS tag_name FROM notes n " +
            "JOIN note_tags t ON n.id = t.note_id WHERE n.id = #{id}")
    NoteWithTags selectWithTags(@Param("id") Long id);
}
```

MP 的 `@Select` 直接写原生 SQL，多表 JOIN 一目了然。JPA 要用 JPQL 或 `nativeQuery=true`，可读性差。

## IService（进阶）

MP 还提供了 `IService` 接口 + `ServiceImpl` 实现类，比 `BaseMapper` 更高层：

```java
@Service
public class NoteService extends ServiceImpl<NoteMapper, Note> {
    // 继承即拥有：list, getById, save, updateById, removeById, page ...
    // 比 BaseMapper 多了批量操作、Lambda 查询等
}
```

Day 5 学习用 `BaseMapper` 足够，`IService` 是进阶用法。

## Day 5 实战：JPA + MP 共存

同一个项目里 JPA 和 MyBatis-Plus 可以共存，共享同一个 DataSource：

1. `Note` 实体同时加 `@Entity`（JPA）和 `@TableName`（MP）
2. `NoteRepository`（JPA）和 `NoteMapper`（MP）并存
3. 需要手动配置 `SqlSessionFactory`（JPA 自动配置会抢占）
4. `PaginationInnerInterceptor` 必须注册到 `SqlSessionFactory`

**踩坑**：MyBatis-Plus 3.5.9 把 `PaginationInnerInterceptor` 从 `mybatis-plus-extension` 移到了 `mybatis-plus-jsqlparser` 独立模块。

## 面试怎么问

- "JPA 和 MyBatis-Plus 的区别？" → 面向对象 vs 面向 SQL，国际 vs 国内
- "MyBatis-Plus 的 BaseMapper 提供了哪些方法？" → selectById, insert, updateById, deleteById, selectList, selectPage
- "复杂查询选哪个？" → MP 直接写原生 SQL，JPA 要绕 JPQL
- "两者能共存吗？" → 能，共享 DataSource，需手动配置 SqlSessionFactory

## 关联

- 对比：[[Spring Data JPA 与 JpaRepository]]
- 分页：[[分页查询（JPA vs MyBatis-Plus）]]
- 实体注解对比：[[JPA 实体注解]]
- 配置：[[application.properties 配置]]
