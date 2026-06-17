# equals 与 hashCode

> HashSet/HashMap 判断"两个对象是否相同"靠这两个方法;默认按内存地址比,所以放自定义对象去重时通常要重写。

## 它解决什么问题
让"内容相同的对象"被集合正确识别为同一个,实现去重 / 按 key 查找。

## 核心机制(HashSet/HashMap 的判同两步)
1. **先比 `hashCode()`**:把对象散列到某个"桶"。哈希值不同 → 直接认定不同,不再看内容。
2. **同一个桶内再比 `equals()`**:确认内容是否真的相等。

## 为什么要重写
- `Object` 默认的 equals/hashCode 按**内存地址**比 → 只有"同一个对象本身"才算相等。
- 两个内容相同但 new 出来的不同对象 → 默认哈希值不同 → 被当成两个东西 → **去重失败**。
  ```java
  var a = new Doc("报销","财务");
  var b = new Doc("报销","财务");   // 内容同,对象不同
  // 未重写: set.add(a); set.add(b); size==2 ❌
  // 重写后(按字段比): size==1 ✅
  ```

## 铁律:必须同时重写
- 只重写 equals 不重写 hashCode → 内容相等的对象哈希值仍不同 → 分到不同桶 → 根本走不到 equals 比较 → **仍然去重失败**。
- 规则:**equals 相等的对象,hashCode 必须相等**;所以两者要一起重写。

## record 的关系
[[record]] 自动按**字段值**生成 equals/hashCode,所以内容相同的两个 record 天然被判为相等、能正确去重,且不会犯"只重写一个"的错。这正是优先用 record 装数据的原因。

## 面试怎么问(高频)
- "为什么重写 equals 要同时重写 hashCode?"
- "HashSet 怎么判断重复?" → 先 hashCode 定桶,再 equals 比内容。

## 关联
- 依赖方:[[Java 集合框架]](Set/Map 去重靠它)
- 体现:[[record]]
