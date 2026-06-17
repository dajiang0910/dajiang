# 踩坑-PowerShell 吞 -D 参数

> Windows 下 `mvn -Dxxx=yyy` 在 PowerShell 里会被拆散,导致 `Unknown lifecycle phase` 报错;给整个 `-D` 参数加双引号即可。

## 现象
```
mvn -q compile exec:java -Dexec.mainClass=com.journey.Day1
[ERROR] Unknown lifecycle phase ".mainClass=com.journey.Day1".
```
`-Dexec` 被 PowerShell 吞掉,只剩 `.mainClass=...` 被 Maven 当成生命周期阶段。

## 原因
PowerShell 解析裸的 `-D...=...` 参数时会把它拆散,不能正确整体传给 Maven。**代码本身没问题**,是终端参数解析问题。

## 解决(任选)
1. **加双引号**(三种终端通用):
   ```bash
   mvn -q compile exec:java "-Dexec.mainClass=com.journey.Day1"
   ```
2. **直接 IDEA 点绿色三角 ▶ 运行 main**(日常推荐,IDEA 自己拼参数)。
3. 把 IDEA 默认终端换成 **Git Bash**(Settings → Tools → Terminal → Shell path),bash 不拆参数。

## 教训
Windows 命令行传带 `=` 的 `-D` 参数,习惯性加双引号最稳。

## 关联
- 上位:[[Maven 工程结构]]
