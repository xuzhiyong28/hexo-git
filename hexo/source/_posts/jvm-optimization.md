---
title: JVM参数详解和调优
tags:
  - java并发
categories:  java
description : JVM参数详解和调优
date: 2020-10-07 15:25:44
---
## JVM常用参数
### 推大小设置

```shell
-Xms: 初始堆大小
-Xmx: 最大堆大小
-XX:NewSize=n: 设置年轻代大小
-XX:NewRatio=n: 设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n: 年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-XX:MetaspaceSize=n 元空间初始大小
-XX:MaxMetaspaceSize=n 元空间最大大小
```

典型设置

```shell
java -Xmx3550m -Xms3550m -Xss128k -XX:NewRatio=4 -XX:SurvivorRatio=4 -XX:MaxTenuringThreshold=0
#-XX:NewRatio=4:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5
#-XX:SurvivorRatio=4：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6
#-XX:MaxTenuringThreshold=0： 设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。(正常可以不设置)
```

### 回收器

xxx

### 日志和辅助设置

日志相关参考另外一篇博客：[GC日志详解](https://xuzyblog.top/2019/02/18/gc-log/)

## JVM调优

### 什么时候才需要调优？

**如果各项参数设置合理，系统没有超时日志出现，GC频率不高，GC耗时不高，那么没有必要进行GC优化；如果GC时间超过1-3秒，或者频繁GC，则必须优化。**

如果满足下列指标，则一般不需要GC优化（具体情况具体分析）

- Minor GC执行时间不到100ms
- Minor GC执行不频繁，约5秒一次
- Full GC执行时间不到1s
- Full GC执行频率不算频繁，不低于1分钟1次

### 调优目标

- GC的时间足够的小
- GC的次数足够的少
- 发生Full GC的周期足够的长

调优准则

- 根据机器情况和服务需求，具体问题具体分析
- 多数的Java应用不需要在服务器上进行GC优化
- 减少使用全局变量和大对象 (代码层面)
- 调整新生代的大小
- 设置老年代的大小（老年代：新生代=2：1）
- 选择合适的GC收集器
- 设置合适线程堆栈大小