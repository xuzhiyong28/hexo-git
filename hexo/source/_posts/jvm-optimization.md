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
-Xmn512M:新生代的大小
-XX:newSize:新生代初始化内存的大小(注意：该值需要小于-Xms的值)
-XX:MaxnewSize:新生代可被分配的内存的最大上限(注意：该值需要小于-Xmx的值)
-Xmn:年轻代的大小(eden + 2 survivor,设置了这就不用设置-XX:NewSize，-XX:MaxNewSize)
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

### 如何确定新生代和老年代的比例？

- IO交互性系统，互联网上目前大部分的服务都属于该类型，例如分布式 RPC、MQ、HTTP 网关服务等，对内存要求并不大，大部分对象在 TP9999 的时间内都会死亡， Young 区越大越好。
- 内存计算性系统，主要是分布式数据计算 Hadoop，分布式存储 HBase、Cassandra，自建的分布式缓存等，对内存要求高，对象存活时间长，Old 区越大越好。

## 常见知识点

### 空间分配担保

空间担保指的是老年代进行空间分配担保。在发生Minor GC前，JVM会先检查**老年代最大可用的连续空间是否大于新生代所有对象的总空间**。

- 如果大于，则此次**Minor GC是安全的**。
- 如果小于，则虚拟机会查看**HandlePromotionFailure**设置值是否允许担保失败。如果HandlePromotionFailure=true，那么会继续检查老年代最大可用连续空间是否大于**历次晋升到老年代的对象的平均大小**，如果大于，则尝试进行一次Minor GC，但这次Minor GC依然是有风险的；如果小于或者HandlePromotionFailure=false，则改为进行一次Full GC。

**为什么要进行空间担保？**

是因为新生代采用**复制收集算法**，假如大量对象在Minor GC后仍然存活（最极端情况为内存回收后新生代中所有对象均存活），而Survivor空间是比较小的，这时就需要老年代进行分配担保，把Survivor无法容纳的对象放到老年代。**老年代要进行空间分配担保，前提是老年代得有足够空间来容纳这些对象**，但一共有多少对象在内存回收后存活下来是不可预知的，**因此只好取之前每次垃圾回收后晋升到老年代的对象大小的平均值作为参考**。使用这个平均值与老年代剩余空间进行比较，来决定是否进行Full GC来让老年代腾出更多空间。

![空间担保机制](jvm-optimization/1.png)

### 什么情况对象直接进老年代

- 在执行YGC时，To Survivor区不足以存放存活的对象，对象会直接进入到老年代。
- 长期存活的对象将进入老年代。经过多次YGC后，如果存活对象的年龄达到了设定阈值，则会晋升到老年代中，通过-XX：MaxTenuringThreshold = 15设置。
- 大对象直接进入老年代，通过-XX：PretenureSizeThreshold参数设置，如果大于这个设定值则直接存入老年代，
- 动态年龄判定规则，To Survivor区中相同年龄的对象，如果其大小之和占到了 To Survivor区一半以上的空间，那么大于此年龄的对象会直接进入老年代，而不需要达到默认的分代年龄。

### 什么情况下会触发FGC

- 当晋升到老年代的对象大于了老年代的剩余空间时，就会触发FGC（Major GC），**FGC处理的区域同时包括新生代和老年代**。
- 老年代的内存使用率达到了一定阈值（可通过参数调整），直接触发FGC(-XX:CMSInitiatingOccupancyFraction=70 是指设定CMS在对内存占用率达到70%的时候开始GC）。
- 空间分配担保：在YGC之前，会先检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。如果小于，说明YGC是不安全的，则会查看参数 HandlePromotionFailure 是否被设置成了允许担保失败，如果不允许则直接触发Full GC；如果允许，那么会进一步检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果小于也会触发 Full GC。
- Metaspace（元空间）在空间不足时会进行扩容，当扩容到了-XX:MetaspaceSize 参数的指定值时，也会触发FGC。
- System.gc() 或者Runtime.gc() 被显式调用时，触发FGC。（有些第三方插件会有用这个函数）

### 三种内存回收机制优缺点

对于 Java 的内存回收机制我们主要分为 3 种：

- 标记-清除算法
  - 不足：效率不高，容易产生内存碎片。
- 复制算法
  - 效率高，复制成本大。
  - 不足：如果是 1：1 模型，占用内存。
  - 分配担保策略
- 标记-整理算法
  - 因为老年代的对象生存率高，所以使用复制算法效率会很差，就提出了这个标记-整理算法。

### CMS老年代垃圾回收器详解

CMS是一款并发、使用*<font color=red>标记-清除</font>*算法的老年代垃圾回收器。针对老年代进行回收的GC。<font color=red>以获取最短回收停顿时间【也就是指Stop The World的停顿时间】</font>为目标，多数应用于互联网站或者B/S系统的服务器端上。

**CMS四个过程**：

CMS基于**标记-清除**算法实现。

- 初始标记(<font color=red>Stop The World</font>)。标记一下GC Roots能直接关联到的对象。这一步主要做两件事：
- - 从GcRoots直接可达的老年代对象
  - 遍历新生代对象，标记可达的老年代对象
- 并发标记。遍历初始标记阶段标记出来的存活对象，然后继续递归标记这些对象可达的对象。**在运行期间可能发生新生代的对象晋升到老年代、或者是直接在老年代分配对象、或者更新老年代对象的引用关系等等，对于这些对象，都是需要进行重新标记的，否则有些对象就会被遗漏，发生漏标的情况。**为了提高重新标记的效率，该阶段会把上述对象所在的Card(卡表)标识为Dirty(脏)，后续只需扫描这些Dirty Card的对象，避免扫描整个老年代。
- 并发预清理。*CMSPrecleaningEnabled*控制是否需要并发预清理，默认启用。**该阶段要尽最大的努力去处理那些在并发阶段被应用线程更新的老年代对象，这样在暂停的重新标记阶段就可以少处理一些，暂停时间也会相应的降低。**
- 重新标记。为了修正并发标记期间因用户程序继续动作而导致标记产生变动的那一部分对象的标记记录 --- <font color=red>Stop The World</font>
- 并发清除。

**优缺点**

- 并发收集、低停顿。
- CMS收集器对处理器资源非常敏感。在并发阶段，它虽然不会导致用户线程停顿，但却会因为占用了一部分线程（或者说处理器的算能力）而导致应用程序变慢，降低总吞吐量。
- CMS收集器无法处理浮动垃圾（因为CMS是支持并发的原因，所以允许在过程中用户线程也在运行，会导致判断不准确。 例如可能在判断完成之后在清除之前这个对像已经变成了垃圾对象，所以有可能本该此垃圾被回收但是没有被回收，只能等待下一次GC再将该对象回收）。

**几个优化的参数**

- -XX:+UseCMSInitiatingOccupancyOnly 和 -XX：CMSInitiatingOccupancyFraction的值来用来设置CMS收集器老年代占用百分之多少后执行FullGC，降低内存回收频率，获取更好的性能。
- -XX:+UseCMSCompactAtFullCollection默认开启，要进行Full GC的时候进行内存碎片整理。
- -XX:CMSFullGCsBeforeCompaction 每隔多少次不压缩的Full GC后，执行一次带压缩的Full GC。
- CMSScavengeBeforeRemark 如果开启这个参数，会在进入重新标记阶段之前强制触发一次minor gc。
- CMSMaxAbortablePrecleanTime = 5 表示并发预清理阶段持续5秒还没等到minor gc就中断并发预清理阶段。

**CMS并发模式失败（Concurrent mode failure）和晋升失败**

- 并发模式失败，CMS的目标就是在回收老年代对象的时候不要停止全部应用线程，在并发周期执行期间，用户的线程依然在运行，如果这时候如果应用线程向老年代请求分配的空间超过预留的空间（担保失败），就回触发concurrent mode failure，然后CMS的并发周期就会被一次Full GC代替。
- 晋升失败，新生代做minor gc的时候，需要CMS的担保机制确认老年代是否有足够的空间容纳要晋升的对象，担保机制发现不够，则报concurrent mode failure，如果担保机制判断是够的，但是实际上由于碎片问题导致无法分配，就会报晋升失败。
- 永久代空间（或Java8的元空间）耗尽，默认情况下,CMS不会对永久代进行收集，一旦永久代空间耗尽，就回触发Full GC。

针对并发模式失败的调优 ：

- 想办法增大老年代的空间，增加整个堆的大小，或者减少年轻代的大小。
- 以更高的频率执行后台的回收线程，即提高CMS并发周期发生的频率，例如调低*CMSInitiatingOccupancyFraction*的值。引用《Java性能权威指南》（**对特定的应用程序，该标志的更优值可以根据 GC 日志中 CMS 周期首次启动失败时的值得到。具体方法是，在垃圾回收日志中寻找并发模式失效，找到后再反向查找 CMS 周期最近的启动记录，然后根据日志来计算这时候的老年代空间占用值，然后设置一个比该值更小的值。**）
- 增多回收线程的个数CMS默认的垃圾收集线程数是*（CPU个数 + 3）/4*。-XX:ParallelCMSThreads用来设置线程数量。

### JVM动态年龄判断

-XX:MaxTenuringThreshold=3，该参数主要是控制新生代需要经历多少次GC晋升到老年代中的最大阈值。但是不一定是这个阈值才晋升到老年代。因为有动态年龄判断(**Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄**)

### 垃圾回收图解

![垃圾回收过程](jvm-optimization/3.png)

### JVM内存模型

![jvm内存模型](jvm-optimization/4.png)

### 垃圾收集器

![]()![5](jvm-optimization/5.png)

### JAVA内存对象布局

![JAVA内存对象布局](jvm-optimization/6.png)

```java
package xzy;
import org.openjdk.jol.info.ClassLayout;
public class OtherTest {
    public static void main(String[] args){
        Object obj = new Object();
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
//==========================================无指针压缩=========================
# -XX:-UseCompressedOops
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)对象头                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)对象头                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)Class Pointer                           00 1c 95 1e (00000000 00011100 10010101 00011110) (513088512)
     12     4        (object header)Class Pointer                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
//==========================================指针压缩=========================
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)对象头                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)对象头                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)Class Pointer                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)对其填充
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
    
    
public class OtherTest {
    public static void main(String[] args){
        Object[] obj = new Object[10];
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
    }
}
//===============================无指针压缩数组===========================
[Ljava.lang.Object; object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)对象头                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)对象头                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)Class Pointer                           60 c7 b1 1e (01100000 11000111 10110001 00011110) (514967392)
     12     4                    (object header)Class Pointer                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     4                    (object header)Class Pointer                           0a 00 00 00 (00001010 00000000 00000000 00000000) (10)
     20     4                    (alignment/padding gap)对齐填充                  
     24    80   java.lang.Object Object;.<elements>数组长度(8 * 10)                        N/A
Instance size: 104 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
//===========================指针压缩数组==============================
 [Ljava.lang.Object; object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)对象头                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)对象头                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)Class Pointer                           3c 23 00 f8 (00111100 00100011 00000000 11111000) (-134208708)
     12     4                    (object header)对齐填充                           0a 00 00 00 (00001010 00000000 00000000 00000000) (10)
     16    40   java.lang.Object Object;.<elements>数组长度(4 * 10)                        N/A
Instance size: 56 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

### 三色标记法

实现并发标记的算法就是三色标记法，三色标记法最大的特点就是可以异步执行，从而可以以中断时间极少的代价或者完全没有中断来进行整个GC。

- 白色：尚未被GC访问过的对象，如果全部标记已完成依旧为白色的，称为不可达对象，既垃圾对象。
- 黑色：本对象已经被GC访问过，且本对象的子引用对象也已经被访问过了。
- 灰色：本对象已访问过，但是本对象的子引用对象还没有被访问过，全部访问完会变成黑色，属于中间态。

**标记过程**

1. 在GC并发标记刚开始时，所以对象均为白色集合。
2. 将所有GCRoots直接引用的对象标记为灰色集合。
3. 判断若灰色集合中的对象不存在子引用，则将其放入黑色集合，若存在子引用对象，则将其所有的子引用对象放入灰色集合，当前对象放入黑色集合。