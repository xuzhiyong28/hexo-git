---
title: JAVA线程池详解
tags:
  - java并发
categories:  java
description : 详解线程池用法和原理
date: 2018-01-18 21:10:00
---
## 为什么要使用线程池?
多线程技术主要解决处理器单元内多个线程执行的问题，它可以显著减少处理器单元的闲置时间，增加处理器单元的吞吐能力。使用多线程技术完成一个任务所需的时间包括。
- T1 创建线程时间
- T2 执行线程任务的时间
- T3 销毁线程的时间

如果：T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。不单单如此，合理的线程池帮我们管理系统线程资源防止由于线程太多导致的系统崩溃问题等。
所以,**线程池有如下作用**:
- 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务
- 可根据系统的承受能力调整线程池中工作线线程的数目防止因为消耗过多的内存而把服务器累趴下

## 线程池详解
### 继承体系
![](executorservice/1.png)
Executor为线程池的顶级接口，声明了<font color=#FF0000 >execute</font>方法。
Executors为创建线程池的工具类，提供了多种方式的线程池创建方法。

### 核心参数
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| corePoolSize    | 核心线程数。当线程数小于该值时，线程池会优先创建新线程来执行新任务 |
| maximumPoolSize | 线程池所能维护的最大线程数                                   |
| keepAliveTime   | 空闲线程的存活时间                                           |
| workQueue       | 任务队列，用于缓存未执行的任务                               |
| threadFactory   | 线程工厂。可通过工厂为新建的线程设置更有意义的名字           |
| handler         | 拒绝策略。当线程池和任务队列均处于饱和状态时，使用拒绝策略处理新任务。默认是 AbortPolicy，即直接抛出异常 |



### 线程池执行逻辑

![](executorservice/2.png)
假设目前定义了一个线程池
corePoolSize = 2 
keepAliveTime = 10ms
maximumPoolSize = 3
workQueue = ArrayBlockingQueue(长度为3)

执行步骤如下:
1. 首先进来一个任务A,在线程池没有任务在执行,正在运行线程 <  核心线程数,建一个线程执行任务
2. 又来一个任务B,此时正在运行的线程数为1.正在运行线程 <  核心线程数,则继续创建一个线程执行任务
3. 又来一个任务C,此时正在运行线程 > 核心线程数,将任务放到workQueue等待执行。
4. 此时又来了任务DEF.....,由于队列只允许存3个,DE进入到队列等待
5. 此时F没有进入队列, 正在运行线程 < 最大线程数,则任务F允许创建一个线程执行。
6. 此时任务G到来，由于AB任务还没结束，workQueue存放了CDE任务等待执行,此时正在运行线程 =最大线程数,则G被拒绝，执行对应的拒绝策略。
7. 此时任务A执行完毕,任务C则从workQueue取出,沿用刚开始创建的线程执行任务。

**keepAliveTime** : 由于任务F是额外创建的一个线程去执行，线程池在空闲时只能保持 核心线程数个线程，所以当任务F执行完后，在keepAliveTime时间内如果没有新的任务进来导致正在运行线程 > 核心线程数 & workQueue已满 这两个条件，则对应的线程会被销毁。

线程创建规则如下:

| 序号 | 条件                                                        | 动作             |
| ---- | ----------------------------------------------------------- | ---------------- |
| 1    | 线程数 < corePoolSize                                       | 创建新线程       |
| 2    | 线程数 ≥ corePoolSize，且 workQueue 未满                    | 缓存新任务       |
| 3    | corePoolSize ≤ 线程数 ＜ maximumPoolSize，且 workQueue 已满 | 创建新线程       |
| 4    | 线程数 ≥ maximumPoolSize，且 workQueue 已满                 | 使用拒绝策略处理 |

从线程池执行逻辑总结 :

- 任务进来时首先先在核心线程中执行，如果核心线程满则放到队列中，如果队列满且还没到最大线程数则可以额外创建线程执行(这个额外的线程是会被回收的)
- 核心线程不会被回收，额外的线程会被回收，如果设置了allowCoreThreadTimeOut为true，则核心线程数也可以被回收，但一般不这样做
- 如果workQueue是无界队列，则keepAliveTime将不起作用

### 排队策略
当线程数 ≥ corePoolSize时，任务会存到队列中排队，有3中类型的容器可供使用，分别是同步队列，有界队列和无界队列。对于有优先级的任务，这里还可以增加优先级队列。

| 实现类                | 类型       | 说明                                                         |
| --------------------- | ---------- | ------------------------------------------------------------ |
| SynchronousQueue      | 同步队列   | 该队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直阻塞 |
| ArrayBlockingQueue    | 有界队列   | 基于数组的阻塞队列，按照 FIFO 原则对元素进行排序             |
| LinkedBlockingQueue   | 无界队列   | 基于链表的阻塞队列，按照 FIFO 原则对元素进行排序             |
| PriorityBlockingQueue | 优先级队列 | 具有优先级的阻塞队列                                         |

### 拒绝策略
线程数量大于等于 maximumPoolSize，且 workQueue 已满，则使用拒绝策略处理新任务。Java 线程池提供了4中拒绝策略实现类

| 实现类              | 说明                                          |
| ------------------- | --------------------------------------------- |
| AbortPolicy         | 丢弃新任务，并抛出 RejectedExecutionException |
| DiscardPolicy       | 不做任何操作，直接丢弃新任务                  |
| DiscardOldestPolicy | 丢弃队列队首的元素，并执行新任务              |
| CallerRunsPolicy    | 由调用线程执行新任务                          |

## 几种线程池
Executors为创建线程池的工具类，提供了多种方式的线程池创建方法。
### newFixedThreadPool
构建包含固定线程数的线程池，默认情况下，空闲线程不会被回收。
从参数可以看出核心线程数 = 最大线程数 / 采用无界队列。
```
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
**特点与隐患**
这种方式采用的是无界队列，如果任务执行时间长且任务量多将会出现任务堆积在队列中导致系统内存飙升，内存溢出等问题
### newCachedThreadPool
构建线程数不定的线程池，线程数量随任务量变动，空闲线程存活时间超过60秒后会被回收。
**SynchronousQueue**特点是队列不存储元素，每个插入操作必须等待另一个线程调用移除操作，否则插入操作会一直阻塞。

```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
**特点与隐患**
队列相当于不排队，每次来一个任务就创建一个线程执行。当对应的任务执行完以后对应的线程如果在60秒内没有没有其他任务继续复用则会销毁。
**推荐使用这种线程池**。能够自动在并发量不多的时候自动销毁执行完的线程。但是最大线程数为无穷大,没有控制线程数也容易造成当并发量特别高时不加以控制导致线程数飙升
### newSingleThreadExecutor
构建线程数为1的线程池，等价于 newFixedThreadPool(1) 所构造出的线程池
```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

```
**特点与隐患**
一次只能一个线程在运行，只会创建一个线程进行复用，采用无界队列容易造成内存飙升
### newScheduledThreadPool
创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。
```
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
   public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }


```

## 代码分析
