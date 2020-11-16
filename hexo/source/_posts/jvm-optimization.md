---
title: JVM参数详解和调优
tags:
  - java并发
categories:  java
description : JVM参数详解和调优
date: 2020-05-07 15:25:44
---
## JVM常用参数
### 推和栈大小设置

```shell
-Xms1024M: 初始堆大小
-Xmx1024M: 最大堆大小
-XX:NewSize=n: 设置年轻代大小
-XX:NewRatio=n: 设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n: 年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-Xss256K: 栈大小设置
-XX:MetaspaceSize=n 元空间初始大小
-XX:MaxMetaspaceSize=n 元空间最大大小
```
<!--more-->
典型设置

```shell
java -Xmx3550m -Xms3550m -Xss128k -XX:NewRatio=4 -XX:SurvivorRatio=4 -XX:MaxTenuringThreshold=0
#-XX:NewRatio=4:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5
#-XX:SurvivorRatio=4：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6
#-XX:MaxTenuringThreshold=0： 设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。(正常可以不设置)
```

### 回收器

JVM垃圾回收器按照分类可以分成：串行收集器，并行收集器，并发收集器。

- 串行收集器：一个GC线程进行回收，会暂停所有用户线程，不符合服务器环境。
- 并行收集器：多个GC线程进行回收，会暂停所有用户线程，适用于大数据，科学计算处理场景。
- 并发收集器(CMS)：用户线程和GC线程同事执行，不会暂停用户线程，适用于对响应时间要求高的场景。

如何选择垃圾收集器，可以通过如下几点。

1、单CPU小内存 : -XX:+UseSerialGC

2、多CPU，需要大量运算：-XX:+UseParallelGC  -XX:+UseParallelOldGC

3、多CPU，要求快速响应：-XX:+UseParNewGC  -XX:+UseConcMarkSweepGC

在C/S架构中往往我们关注的是**响应时间**。并发收集器主要是保证系统的响应时间，减少垃圾收集时的停顿时间。所以我们一般使用的是并发收集器，也就是CMS。配置参考如下

```shell
-Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+UseCMSCompactAtFullCollection
#-XX:+UseConcMarkSweepGC： 设置年老代为并发收集
#-XX:+UseParNewGC: 设置年轻代为并行收集。
#-XX:+UseCMSCompactAtFullCollection 打开对年老代的压缩。可能会影响性能，但是可以消除碎片
```

### 日志和辅助设置

日志相关参考另外一篇博客：[GC日志详解](https://xuzyblog.top/2019/02/18/gc-log/)

另外一些辅助类的配置，笔者项目中配置的一些优化。

这里顺便说下 : **-XX:+ 这种表示开启什么配置，-XX:- 表示关闭什么配置**

```shell
-XX:+DisableExplicitGC #加了这个配置后，再代码上使用System.gc()将不会生效，System.gc()会出发FullGC
-XX:+UseCompressedOops #开启压缩指针，例如压缩指针对象头。开启后可以节省一定的内存空间
-XX:+CMSParallelRemarkEnabled #降低标记停顿
-XX:CMSInitiatingOccupancyFraction=70 #使用cms作为垃圾回收,使用70％后开始CMS收集
```

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

例如线上排查Full GC次数频繁。这就要知道什么时候对象会放到老年代。

- YGC时，To Survivor区不足以存放存活的对象，对象会直接进入到老年代。这种情况可以适当加大Survivor区的大小。
- 经过多次YGC后，如果存活对象的年龄达到了设定阈值，则会晋升到老年代中。这种情况属于对象存活太久，如果大对象在业务上不需要使用那么久最好能够用完即删。
- 动态年龄判定规则，To Survivor区中相同年龄的对象，如果其大小之和占到了 To Survivor区一半以上的空间，那么大于此年龄的对象会直接进入老年代，而不需要达到默认的分代年龄。这种情况也是可以适当更改Survivor大小。