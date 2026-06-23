# Jackson

> Java 最主流的 JSON 解析库(`com.fasterxml.jackson.databind.ObjectMapper`),类比 Python 的 `json`/Pydantic。

## 它解决什么问题
把 JSON 字符串转成 Java 对象(LLM 输出的结构化数据、API 响应)。

## 核心要点
- 入口:`ObjectMapper`(单例复用,创建它有开销)
- 两种读法:

| 方法                                 | 返回              | 适合场景                  |
| ---------------------------------- | --------------- | --------------------- |
| `readTree(json)`                   | `JsonNode`(通用树) | 结构不确定、字段可能缺失          |
| `readValue(json, XxxRecord.class)` | 直接转对象           | 结构已知、要拿到 record/POJO  |

- 配套取值:`node.get("字段").asDouble() / .asInt() / .asText()`,字段缺失返回 `0` 或 `null`,**不抛异常**

## Maven 依赖
```xml
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.17.2</version>
</dependency>
```

## 实战代码(Day5)
```java
ObjectMapper mapper = new ObjectMapper();
JsonNode root = mapper.readTree(response.body());
double temp = root.get("current").get("temperature_2m").asDouble();
```

## 面试怎么问
- "readTree 和 readValue 区别?" → 通用树 vs 直接转对象
- "Jackson 线程安全吗?" → **ObjectMapper 线程安全**,可作单例

## 关联
- 配合:[[HttpClient]](API 响应解析)
- 数据载体:[[record]](readValue 经常直接转 record,访问器无 get 前缀)
- 进阶:W4 会用 `BeanOutputConverter` 把 LLM 输出转 record
