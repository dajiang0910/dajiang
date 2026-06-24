# application.properties 配置

> Spring Boot 的核心配置文件。数据源、JPA、服务器端口等全在这里配。支持 `.properties` 和 `.yml` 两种格式。

## Day 4 实战配置

```properties
# === 数据源 ===
spring.datasource.url=jdbc:h2:mem:notesdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# === JPA / Hibernate ===
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# === H2 控制台 ===
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

## 配置分类速查

### 数据源（datasource）

| 配置项 | 作用 | 示例 |
|---|---|---|
| `spring.datasource.url` | 数据库连接地址 | `jdbc:h2:mem:notesdb` |
| `spring.datasource.driver-class-name` | JDBC 驱动类 | `org.h2.Driver` |
| `spring.datasource.username` | 用户名 | `sa` |
| `spring.datasource.password` | 密码 | 留空 / `root` |

### JPA（jpa.hibernate）

| 配置项 | 作用 | 示例 |
|---|---|---|
| `spring.jpa.hibernate.ddl-auto` | DDL 自动生成策略 | `update` |
| `spring.jpa.show-sql` | 控制台打印 SQL | `true` |
| `spring.jpa.properties.hibernate.format_sql` | 格式化 SQL 输出 | `true` |

### ddl-auto 详解

| 值 | 行为 | 场景 |
|---|---|---|
| `create` | 启动 DROP + CREATE，**数据清空** | 纯本地从零开始 |
| `create-drop` | 启动建表，关闭时删表 | 集成测试 |
| `update` | 只加新表/字段，不删不改旧字段 | 快速原型开发 |
| `validate` | 只校验表结构，不改表 | **生产推荐** |
| `none` | 啥也不做 | 生产用 Flyway/Liquibase |

**重要**：`update` 不能处理列重命名、类型修改等场景。生产环境用 `validate` 或 `none`，数据库变更交给 Flyway 等迁移工具。

### H2 控制台（h2.console）

| 配置项 | 作用 | 默认值 |
|---|---|---|
| `spring.h2.console.enabled` | 开启 H2 网页控制台 | `false` |
| `spring.h2.console.path` | 控制台路径 | `/h2-console` |
| `spring.h2.console.settings.web-allow-others` | 允许远程访问 | `false` |

## .properties vs .yml 格式对比

```properties
# application.properties
spring.datasource.url=jdbc:h2:mem:notesdb
spring.jpa.hibernate.ddl-auto=update
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:h2:mem:notesdb
  jpa:
    hibernate:
      ddl-auto: update
```

两种格式等价，选哪种都行。yml 层级更清晰，properties 更简单直接。

## 多环境配置（进阶）

```
src/main/resources/
├── application.properties          # 公共配置
├── application-dev.properties      # 开发环境
├── application-prod.properties     # 生产环境
```

```properties
# application.properties 激活哪个环境
spring.profiles.active=dev
```

不同环境可以配不同的数据库、端口、日志级别。

## 面试怎么问

- "ddl-auto 的几个值区别？" → create（删表重建）/ update（只加不删）/ validate（只校验）/ none（不操作）
- "生产环境用哪个？" → validate 或 none，数据库变更用 Flyway
- "怎么多环境配置？" → application-{profile}.properties + spring.profiles.active

## 关联

- 数据源：[[H2 内存数据库]]
- ORM：[[Spring Data JPA 与 JpaRepository]] + [[JPA 实体注解]]
- 进阶：Spring Profiles 多环境
