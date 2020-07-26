---
title: Future、FutureTask实现原理浅析
tags:
  - java
  - java并发
categories:  java
description : Future、FutureTask实现原理浅析
date: 2019-07-18 10:00:00
---
## 什么是Future、FutureTask
future在字面上表示未来的意思，在Java中一般通过继承Thread类或者实现Runnable接口这两种方式来创建多线程，但是这两种方式都有个缺陷，就是不能在执行完成后获取执行的结果。然而JDK提供了一种类似ajax的方式，允许提交任务后去做自己的事，在任务执行完成后可以获得执行的结果。总的来说就是实现"任务的提交"和"任务的执行"相分离。

```
@Test
    public void test2() throws ExecutionException, InterruptedException {
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            TimeUnit.SECONDS.sleep(5);
            return "success";
        });
        futureTask.run();
        System.out.println("=====做自己想做的其他事情====");
        System.out.println(futureTask.get());
    }
```

### FutureTask的继承关系和常用方法

FutureTask继承关系,从继承关系上看，futureTask实现了Future接口和Runnable接口。所以FutureTask实现了表格上方法。

![](futuretask/1.png)

| 方法        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| cancel      | 取消任务                                                     |
| isCancelled | 判断任务是否已取消                                           |
| isDone      | 判断任务是否已结束                                           |
| get         | 以阻塞方式获取任务执行结果，如果任务还没有执行完，调用get（），会被阻塞，直到任务执行完才会被唤醒 |
| run         | 线程执行的方法                                               |

## FutureTask源码解析

### FutureTask构造函数

FutureTask支持传入Runnable和Callable，但是Runable并不支持返回值。所以在FutureTask(Runnable runnable, V result)构造函数中使用了Executors.callable(runnable, result)方法采用适配器模式将Runnable转成Callable。

![](futuretask/2.png)

```java
public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
}
static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
}
```

通过适配器模式将Runnable转成Callable。

### FutureTask的状态变量

```java
//表示当前task状态
private volatile int state;
//当前任务尚未执行
private static final int NEW          = 0;
//当前任务正在结束，稍微完全结束，一种临界状态
private static final int COMPLETING   = 1;
//当前任务正常结束
private static final int NORMAL       = 2;
//当前任务执行过程中发生了异常
private static final int EXCEPTIONAL  = 3;
//当前任务被取消
private static final int CANCELLED    = 4;
//当前任务中断中
private static final int INTERRUPTING = 5;
//当前任务已中断
private static final int INTERRUPTED  = 6;
//用来存储 "用户提供的有实在业务逻辑的" 任务
private Callable<V> callable;
//用来保存异步计算的结果。正常情况保存返回值，非正常情况保存异常
private Object outcome;
//当前任务被线程执行期间，保存当前执行任务的线程对象引用
private volatile Thread runner;
/**
futureTask.get是支持多个线程去调用的，这个变量主要是用来存储调用get方法线程的一个队列。当futureTask.run执行完成后会通过这个变量for循环去通知调用的线程结束阻塞
**/
private volatile WaitNode waiters;

```

### FutureTask主流程

```java
public void run() {
       //如果任务不是NEW状态（如果不是NEW就表示Task已经被执行过或者被取消了）
       //UNSAFE.compareAndSwapObject表示将当前执行run方法的线程通过CAS方式设置到runnerOffset变量
       //CAS的特性，如果runnerOffset = null 则将当前线程设置到runnerOffset,成功返回ture失败返回false
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            //callable就是程序员自己写的逻辑
            Callable<V> c = callable;
            //c!=null 防止程序员没写自己的逻辑
            //为什么又判断了一次? 防止期间有外部任务执行了cancel掉了当前任务
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //执行任务
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //设置失败
                    setException(ex);
                }
                if (ran)
                    //任务执行正常，设置结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

FutureTask#set()设置执行结果函数

```java
    protected void set(V v) {
        //使用CAS，判断当前状态是不是NEW，如果是就设置成COMPLETING。通过CAS保证了只有一个线程
        //去设置结果
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //将结果设置给outcome
            outcome = v;
            //将结果设置给outcome后，马上将状态设置成NORMAL
            //putOrderedInt表示设置值 并且马上写入主存
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            //TODO
            finishCompletion();
        }
    }
```



