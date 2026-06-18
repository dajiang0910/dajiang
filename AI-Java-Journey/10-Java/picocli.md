# picocli

> Java 主流命令行参数解析库,一行注解搞定 `--city 上海` 这类 CLI 入口。Week 1 CLI 工具收官靠它。

## 它解决什么问题
手写 `if (args[0].equals("--city")) ...` 痛点:
- 类型转换手写(字符串 → 数字 / 路径 / 枚举)
- 必填校验手写
- `--help` / `--version` 手写
- 错误信息手写

picocli 用**注解 + 反射**把这些都自动化。

## 核心要点
- **注解驱动**:
  ```java
  @CommandLine.Command(name = "weather", description = "查询城市天气")
  public class Main implements Runnable {
      @Option(names = {"-c", "--city"}, description = "城市名", required = true)
      String city;

      public static void main(String[] args) {
          int exit = new CommandLine(new Main()).execute(args);
          System.exit(exit);
      }

      @Override public void run() { ... }   // 业务逻辑
  }
  ```
- **3 个常用注解**:
  - `@Command` —— 标注类为命令
  - `@Option` —— 标注字段为选项(`-c`、`--city`)
  - `@Parameters` —— 标注字段为位置参数(无 `-` 前缀)
- **必填校验**:`required = true` 缺参自动打印 usage + 退出码非 0
- **异常处理**:业务方法 `run()` 抛异常 → picocli 捕获 → 打印堆栈 + 退出码非 0(shell 脚本友好)
- **--help**:`usageHelp = true` 一行搞定

## Java 里怎么落地
Day 6 `Main.java`:picocli 解析 `--city` → 调 `WeatherService.fetch` → 打印。`run()` 不包 try-catch,异常冒到 picocli。

## 面试怎么问(拓展)
- "CLI 工具有哪些要素?" → 参数解析 + 必填校验 + 帮助文档 + 退出码 + 错误处理。
- "为什么进程退出码重要?" → shell 脚本 `if [ $? -ne 0 ]` 判断成败,CI/CD 也靠它。

## 关联
- 上层:[[WeatherService 设计模式]](CLI 入口只做 I/O,业务下沉)
- 兄弟:[[Maven 工程结构]] 的 `exec-maven-plugin` 配合跑 CLI