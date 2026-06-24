# JPA 实体注解

> JPA 通过注解把 Java 类映射到数据库表。`@Entity` 标记实体，`@Id` 标记主键，`@Column` 标记字段约束。

## 核心注解速查

| 注解 | 作用 | 示例 |
|---|---|---|
| `@Entity` | 标记类为 JPA 实体 | `@Entity` |
| `@Table(name = "notes")` | 指定表名（不写则默认类名） | `@Table(name = "notes")` |
| `@Id` | 标记主键字段 | `@Id` |
| `@GeneratedValue(IDENTITY)` | 主键自增策略 | `@GeneratedValue(strategy = GenerationType.IDENTITY)` |
| `@Column(nullable = false, length = 100)` | 字段约束 | 非空、最大长度 |

## Day 4 实战代码

```java
@Entity
@Table(name = "notes")
public class Note {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 数据库自增
    private Long id;

    @Column(nullable = false, length = 100)              // 非空，最长 100 字符
    private String title;

    @Column(length = 10000)                              // 最长 10000 字符
    private String content;

    @Column(name = "created_at")                         // 映射到 created_at 列
    private LocalDateTime createdAt;

    protected Note() {}                                  // JPA 要求无参构造

    public Note(String title, String content) {
        this.title = title;
        this.content = content;
        this.createdAt = LocalDateTime.now();
    }
    // getter / setter ...
}
```

## @GeneratedValue 四种策略

| 策略 | 行为 | 适用数据库 |
|---|---|---|
| `IDENTITY` | 数据库自增（AUTO_INCREMENT） | MySQL、H2 |
| `SEQUENCE` | 数据库序列 | Oracle、PostgreSQL |
| `TABLE` | 用一张辅助表模拟自增 | 通用 |
| `AUTO` | JPA 自动选择（默认） | 由实现决定 |

**Day 4 用 `IDENTITY`**：H2 和 MySQL 都支持。

## 为什么 JPA 要求无参构造函数？

JPA 从数据库加载数据的过程是**反射**：

```java
// Hibernate 内部伪代码
Note note = Note.class.getDeclaredConstructor().newInstance();  // 调用无参构造
note.setId(rs.getLong("id"));          // 反射设值
note.setTitle(rs.getString("title"));
note.setContent(rs.getString("content"));
```

**关键链路**：
1. 数据库返回的是散装字段（id、title、content）
2. Hibernate 需要先创建一个**空对象**，再逐个字段填充
3. 反射创建对象只能调用**无参构造函数**
4. 如果只有 `Note(String title, String content)`，Hibernate 不知道该传什么参数

### 为什么是 `protected` 不是 `public`？

```java
protected Note() {}   // Hibernate 反射能用（反射无视访问修饰符）
                      // 业务代码不能用（外部 new Note() 编译报错）
```

`protected` 防止业务代码误用无参构造创建"残废"对象（没有 title、没有 content）。

## @Column 常用属性

| 属性 | 作用 | 默认值 |
|---|---|---|
| `name` | 列名（不写则默认字段名） | 字段名 |
| `nullable` | 是否允许 NULL | true |
| `length` | 字段最大长度（String 类型） | 255 |
| `unique` | 是否唯一约束 | false |
| `columnDefinition` | 原始 DDL 定义 | 无 |

## Entity 的 getter/setter 要求

JPA 通过反射读写字段，所以：
- **必须有无参构造**（`protected` 即可）
- **必须有 getter/setter**（Hibernate 用来读写字段值）
- **可以没有 `@Column`**（不写则用默认值，字段名 = 列名）

## 面试怎么问

- "@Entity 的作用？" → 标记类为 JPA 实体，Hibernate 自动映射到数据库表
- "@GeneratedValue 四种策略？" → IDENTITY / SEQUENCE / TABLE / AUTO
- "为什么 JPA 需要无参构造？" → Hibernate 反射创建对象时只能调用无参构造
- "为什么用 protected 不用 public？" → 防止业务代码误用半成品对象

## 关联

- 搭配使用：[[Spring Data JPA 与 JpaRepository]]
- 数据库：[[H2 内存数据库]]
- 配置：[[application.properties 配置]]
- 对比：[[DTO 与 Entity 之分]]（Entity 可变需 getter/setter，DTO 用 record 不可变）
