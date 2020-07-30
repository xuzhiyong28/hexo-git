---
title: Fork/join框架你会用吗？
tags:
  - java并发
categories:  java
description : 详解Fork/join框架
date: 2020-07-30 16:35:57
---
## Fork/Join框架介绍

### 介绍

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。Fork/Join框架要完成两件事情：
- 任务分割：首先Fork/Join框架需要把大的任务分割成足够小的子任务，如果子任务比较大的话还要对子任务进行继续分割
- 执行任务并合并结果：分割的子任务分别放到双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都放在另外一个队列里，启动一个线程从队列里取数据，然后合并这些数据

### 使用场景

ForkJoinPool 不是为了替代 ExecutorService，而是它的补充，在某些应用场景下性能比 ExecutorService 更好。FokJoinPool主要适用于<font color=red>计算密集型的任务</font>。例如对一个大数组求和，排序等场景。

## 如何使用Fork/Join框架？

使用Fork/Join框架，首先需要创建一个ForkJoin任务。该类提供了在任务中执行fork和join的机制。通常情况下我们不需要直接集成ForkJoinTask类，只需要继承它的子类，Fork/Join框架提供了两个子类:

- RecursiveAction 用于没有返回结果的任务
- RecursiveTask 用于有返回结果的任务

### 实例一：大数组求和

对一个大数组进行求和。代码如下

```java
@Test
public void test() {
    long[] numbers = LongStream.rangeClosed(1, 5263).toArray();
    ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
    long start = System.currentTimeMillis();
    long result = new ForkJoinPool().invoke(task);
    System.out.println("结果 = " + result  + ", 耗时 = " + (System.currentTimeMillis() - start) + " ms");
}
```

```java
import java.util.concurrent.RecursiveTask;

public class ForkJoinSumCalculator extends RecursiveTask<Long> {
    //不再将任务分解为子任务的数组大小
    public static final long THRESHOLD = 100;
    private final long[] numbers;
    private final int start; //起始位置
    private final int end; //终止位置
    public ForkJoinSumCalculator(long[] numbers) {
        this(numbers, 0, numbers.length);
    }
    private ForkJoinSumCalculator(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            return computeSequentially();
        }
        ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers,start,start + length / 2);
        ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(numbers, start + length/2, end);
        invokeAll(leftTask,rightTask);
        Long rightResult = rightTask.join();
        Long leftResult = leftTask.join();
        return leftResult + rightResult;
    }

    /**
        求和
    */
    private long computeSequentially() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += numbers[i];
        }
        return sum;
    }
}
```

### 实例二：大数组排序

对大数组进行从小到大排序

```java
@Test
public void test4() {
    int MAX = 10000;
    int inits[] = new int[MAX];
    for (int i = 0; i < inits.length; i++) {
        inits[i] = RandomUtils.nextInt(0, MAX);
    }
    ForkJoinPool pool = new ForkJoinPool();
    ForkJoinArraySortTask task = new ForkJoinArraySortTask(inits);
    int[] result = pool.invoke(task);
    //打印
    Arrays.stream(result).forEach(System.out::println);
}
```

```java
public class ForkJoinArraySortTask extends RecursiveTask<int[]> {
    private int source[];

    public ForkJoinArraySortTask(int[] source) {
        this.source = source;
    }

    @Override
    protected int[] compute() {
        int sourceSize = source.length;
        // 如果条件成立，说明任务中要进行排序的集合还不够小
        if (sourceSize > 10) {
            int midIndex = sourceSize / 2;
            //拆分成两个子任务
            ForkJoinArraySortTask leftTask = new ForkJoinArraySortTask(Arrays.copyOf(source, midIndex));
            ForkJoinArraySortTask rightTask = new ForkJoinArraySortTask(Arrays.copyOfRange(source, midIndex, sourceSize));
            invokeAll(leftTask,rightTask);
            int result1[] = leftTask.join();
            int result2[] = rightTask.join();
            int mer[] = joinInts(result1, result2);
            return mer;
        } else {
            //排序
            return Arrays.stream(source).sorted().toArray();
        }
    }

    /***
     * 这个方法用于合并两个有序集合
     * @param result1
     * @param result2
     * @return
     */
    private int[] joinInts(int[] result1, int[] result2) {
        //现将两个数组合起来
        int[] array = ArrayUtils.addAll(result1,result2);
        //排序并去重
        return Arrays.stream(array).sorted().toArray();
    }
}
```

### 实例三：大Map拆分对每个值加10

对一个Map采用Fork/join实现value+10

```java
@Test
public void test6(){
    Map<String, Integer> myMap = Maps.newHashMap();
    for (int i = 0; i < 526252; i++) {
        myMap.put(String.valueOf(i), 1);
    }
    ForkJoinPool pool = new ForkJoinPool();
    ForkJoinMapDoTask forkJoinMapDoTask = new ForkJoinMapDoTask(myMap);
    Map<String, Integer> invoke = pool.invoke(forkJoinMapDoTask);
    System.out.println(invoke.keySet().size());
    for(Map.Entry<String,Integer> entry : invoke.entrySet()){
        if(entry.getValue() != 11){
            System.out.println("不等于");
        }
    }
}
```

```java
/***
 * 对一个大Map进行分割然后值加上10
 */
public class ForkJoinMapDoTask extends RecursiveTask<Map<String, Integer>> {

    private static final int TASK_COUNT = 5000; //按照5000个拆分

    private Map<String, Integer> myMap;

    public ForkJoinMapDoTask(Map<String, Integer> myMap) {
        this.myMap = myMap;
    }


    @Override
    protected Map<String, Integer> compute() {

        if (myMap.keySet().size() <= TASK_COUNT) {
            Map<String, Integer> resultMap = Maps.newHashMap();
            for (Map.Entry<String, Integer> entry : myMap.entrySet()) {
                resultMap.put(entry.getKey(), entry.getValue() + 10);
            }
            return resultMap;
        } else {
            List<Map<String,Integer>> mapList = ForkJoinMapDoTask.mapChunk(myMap,2);
            ForkJoinMapDoTask leftTask = new ForkJoinMapDoTask(mapList.get(0));
            ForkJoinMapDoTask rightTask = new ForkJoinMapDoTask(mapList.get(1));
            invokeAll(leftTask,rightTask);
            Map<String, Integer> lfetMap = leftTask.join();
            Map<String, Integer> rightMap = rightTask.join();
            Map<String, Integer> resultMap = Maps.newHashMap();
            resultMap.putAll(lfetMap);
            resultMap.putAll(rightMap);
            return resultMap;
        }
    }


    public static <k, v> List<Map<k, v>> mapChunk(Map<k, v> chunkMap, int batch) {
        List<Map<k,v>> resultList = Lists.newArrayList();
        if(MapUtils.isEmpty(chunkMap)){
            return Lists.newArrayList();
        }
        int keyLength = chunkMap.keySet().size();
        int pageSize = keyLength % batch == 0 ? (keyLength / batch) : (keyLength / batch + 1);
        //初始化数据
        for(int i = 0 ; i < batch; i ++){
            resultList.add(Maps.newHashMap());
        }
        int i = 0, count = 0;
        for(Map.Entry<k,v> entry : chunkMap.entrySet()){
            resultList.get(i).put(entry.getKey(),entry.getValue());
            count++;
            if(count % pageSize == 0){
                i++;
            }
        }
        return resultList;
    }
}
```

### Fork/Join框架的实现原理

ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，ForkJoinTask数组负责将存放程序提交给ForkJoinPool，而ForkJoinWorkerThread负责执行这些任务;

#### ForkJoinTask的fork方法的实现原理

当我们调用ForkJoinTask的fork方法时，程序会把任务放在ForkJoinWorkerThread的pushTask的workQueue中，异步地执行这个任务，然后立即返回结果，代码如下

```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

pushTask方法把当前任务存放在ForkJoinTask数组队列里。然后再调用ForkJoinPool的signalWork()方法唤醒或创建一个工作线程来执行任务。代码如下：

```java
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)
            growArray();
    }
}
```

#### ForkJoinTask的join方法的实现原理

oin方法的主要作用是阻塞当前线程并等待获取结果。让我们一起看看ForkJoinTask的join方法的实现，代码如下：

```java
public final V join() {
	int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL){
    	reportException(s);
    }
    return getRawResult();
}
```

它首先调用doJoin方法，通过doJoin()方法得到当前任务的状态来判断返回什么结果，任务状态有4种：已完成（NORMAL）、被取消（CANCELLED）、信号（SIGNAL）和出现异常（EXCEPTIONAL）；
如果任务状态是已完成，则直接返回任务结果；
如果任务状态是被取消，则直接抛出CancellationException；
如果任务状态是抛出异常，则直接抛出对应的异常；
doJoin方法的实现，代码如下：

```java
private int doJoin() {
	int s;
	Thread t;
	ForkJoinWorkerThread wt;
	ForkJoinPool.WorkQueue w;
    return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof 								ForkJoinWorkerThread) ? (w = (wt = 										(ForkJoinWorkerThread)t).workQueue).tryUnpush(this) && (s = 				doExec()) < 0 ? s : wt.pool.awaitJoin(w, this, 0L) : 				externalAwaitDone();
}
```

doExec() :

```java
final int doExec() {
	int s; 
	boolean completed;
	if ((s = status) >= 0) {
		try {
			completed = exec();
		} catch (Throwable rex) {
			return setExceptionalCompletion(rex);
		}
		if (completed){
			s = setCompletion(NORMAL);
		}
	}
	return s;
}
```

在doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完成，如果执行完成，则直接返回任务状态；如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成，则设置任务状态为NORMAL，如果出现异常，则记录异常，并将任务状态设置为EXCEPTIONAL。