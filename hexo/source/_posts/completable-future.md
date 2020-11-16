---
title: CompletableFuture使用详解
tags:
  - java并发
  - java8
categories:  java
description : CompletableFuture使用详解
date: 2019-07-18 10:00:00
---
## CompletableFuture介绍
CompletableFuture是JDK8新出来一种异步组合式编程。
Future接口可以构建异步应用，但依然有其局限性。它很难直接表述多个Future 结果之间的依赖性。实际开发中，我们经常需要达成以下目的：
- 将多个异步计算的结果合并成一个
- 等待Future集合中的所有任务都完成
- Future完成事件（即，任务完成以后触发执行动作）
<!--more-->
## CompletableFuture使用
### 场景一：普通异步使用
CompletableFuture提供了4个方法用来执行异步调用。一组是有返回值的，一组是无返回值的
```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

- runAsync 和 supplyAsync 的区别是有没有返回值
- 没有指定Executor的方法会使用ForkJoinPool.commonPool() 作为它的线程池执行异步代码。如果指定线程池，则使用指定的线程池运行。
- 如何判断是自己传线程池还是用ForkJoinPool线程池呢？
  - ThreadPool：多见于线程并发，阻塞时延比较长的，这种线程池比较常用，一般设置的线程个数根据业务性能要求会比较多
  - ForkJoinPool：特点是少量线程完成大量任务，一般用于非阻塞的，能快速处理的业务，或阻塞时延比较少的

```java
@org.junit.Test
public void runOrsupply() throws ExecutionException, InterruptedException {
    //无返回值
    CompletableFuture<Void> voidCompletableFuture = CompletableFuture.runAsync(new Runnable() {
        @Override
        public void run() {
            System.out.println("!!!");
        }
    } , Executors.newCachedThreadPool());
    voidCompletableFuture.get();
   
    //有返回值
    CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(new Supplier<String>() {
        @Override
        public String get() {
            return "success";
        }
    });
    System.out.println(stringCompletableFuture.get());
}
```

### 场景二：任务执行后回调方法

当CompletableFuture任务完成时或抛出异常时执行特定的回调方法。

- whenComplete 和 whenCompleteAsync 表示在任务完成时执行的方法
- exceptionally表示任务异常时回调方法

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

```java
@org.junit.Test
public void testWhenComplete2() throws ExecutionException, InterruptedException {
    CompletableFuture<String> stringCompletableFuture = CompletableFuture.supplyAsync(new Supplier<String>() {
        @Override
        public String get() {
            int i = 1 / 0; //发生异常
            return "xuzy";
        }
    }).whenComplete(new BiConsumer<String, Throwable>() {
        @Override
        public void accept(String s, Throwable throwable) {
            System.out.println("获取supplyAsync返回的值 = " + s);
        }
    }).exceptionally(new Function<Throwable, String>() {
        @Override
        public String apply(Throwable throwable) {
            //当supplyAsync失败时，捕获异常并返回值
            return "xuzy is fail";
        }
    });
    System.out.println(stringCompletableFuture.get());
}
```

### 场景三：任务执行后对任务结果再进行处理

handle 是执行任务完成时对结果的处理。

```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

```java
@org.junit.Test
public void testHandle() throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(new Supplier<Integer>() {
        @Override
        public Integer get() {
            int i = 1 / 0; //发生异常
            return new RandomUtils().nextInt(0, 10);
        }
    }).handle(new BiFunction<Integer, Throwable, Integer>() {
        @Override
        public Integer apply(Integer integer, Throwable throwable) {
            int result = -1;
            //获取到任务的结果，如果没异常则值*2后返回，如果有异常就打印异常并返回-1
            if (throwable == null) {
                result = integer * 2;
            } else {
                System.out.println(throwable.getMessage());
            }
            return result;
        }
    });
    Integer result = future.get();
    System.out.println(result);
}
```

### 场景四 ： 一个线程依赖另一个线程的结果

当一个线程依赖另一个线程时，可以使用 thenApply 方法来把这两个线程串行化。<font color=red>如果线程A出现异常则不会再执行thenApply</font>。

```java
@org.junit.Test
public void testThenApply() throws ExecutionException, InterruptedException {
    //任务A
    CompletableFuture<Long> futureA = CompletableFuture.supplyAsync(new Supplier<Long>() {
        @Override
        public Long get() {
            return RandomUtils.nextLong(0, 100);
        }
    });
    //任务B
    CompletableFuture<String> futureB = futureA.thenApply(new Function<Long, String>() {
        @Override
        public String apply(Long aLong) {
            long result = aLong * 5;
            return "xuzy-" + result;
        }
    });
    System.out.println(futureB.get());
}
```

场景三handle 和 场景四 thenApply区别：

- handle 是在任务完成后再执行，还可以处理异常的任务
- thenApply 只可以执行正常的任务，任务出现异常则不执行 thenApply 方法

### 场景五：任务A执行完成后消费结果

先来理解下什么是消费结果，看如下代码，原来futureA返回的是Integer类型，后执行后thenAccept后就只能是Void了。其实是将futureA的结果获取后消费了。

```java
@org.junit.Test
public void testThenAccept() throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> futureA = CompletableFuture.supplyAsync(new Supplier<Integer>() {
        @Override
        public Integer get() {
            System.out.println("任务A执行");
            return new Random().nextInt(10);
        }
    });
    CompletableFuture<Void> futureVoid = futureA.thenAccept(new Consumer<Integer>() {
        @Override
        public void accept(Integer integer) {
            System.out.println("获取到任务A的结果 ：" + integer + " , 并消费");
        }
    });
    futureVoid.get();
    
    //类似thenAccept的方法thenRun，但是他不能传递任务的结果
    futureA.thenRun(new Runnable() {
            @Override
            public void run() {
				System.out.println("无法获取返回值直接消费");
            }
    });
}
```

### 场景六：将任务AB合并成一个任务执行

thenCombine 会把 两个 CompletionStage 的任务都执行完成后，把两个任务的结果一块交给 thenCombine 来处理。

thenAcceptBoth 会把 两个 CompletionStage 的任务都执行完成后，然后消费。

thenCombine 和 thenAcceptBoth 区别就在于一个是处理结果，一个是消费(没有返回值)

```java
@org.junit.Test
public void testThenCombine() throws ExecutionException, InterruptedException {
    CompletableFuture<String> future1 = CompletableFuture.supplyAsync(new Supplier<String>() {
        @Override
        public String get() {
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "hello";
        }
    });
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(new Supplier<String>() {
        @Override
        public String get() {
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "world";
        }
    });
    CompletableFuture<String> result = future1.thenCombine(future2, new BiFunction<String, String, String>() {
        @Override
        public String apply(String s, String s2) {
            return s + " " + s2;
        }
    });
    long start = System.currentTimeMillis();
    System.out.println(result.get());
    System.out.println((System.currentTimeMillis() - start) + " ms");
}
```

### 场景七：任务A任务B同时执行，看谁快用谁的结果

两个CompletionStage，谁执行返回的结果快，我就用那个CompletionStage的结果进行下一步的消耗操作。

acceptEither

```java
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

**runAfterEither**

```java
public CompletionStage<Void> runAfterEither(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

acceptEither 和 runAfterEither 的区别也是有没有返回值

<font color=red>代码测试发现，谁快的话会取快的任务作为结果继续进行消耗操作，但是慢的那个任务还是会继续执行并没有被取消</font>

```java
@org.junit.Test
public void testJoinOr() {
    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("任务f1");
        return "f1";
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "f2";
    });
    //这里f1.applyToEither(f2  f1在前面还是f2在前面都无所谓
    CompletableFuture<String> f3 = f1.applyToEither(f2, new Function<String, String>() {
        @Override
        public String apply(String s) {
            return "最终是哪个任务的值呢 ？ 值是 : " + s  + ", 获取后可以继续对值做处理";
        }
    });
    long start = System.currentTimeMillis();
    System.out.println(f3.join());
    System.out.println("耗时:" + (System.currentTimeMillis() - start) + " ms");
    try {
        //这里阻塞20秒发现f1的任务还会继续执行，只是在结果上先取了f2的数据
        TimeUnit.SECONDS.sleep(20);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

```java
@org.junit.Test
public void testrunAfterEither(){
    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("任务f1");
        return "f1";
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "f2";
    });
    CompletableFuture<Void> f3 = f1.runAfterEither(f2, new Runnable() {
        @Override
        public void run() {
            System.out.println("f3");
        }
    });
    long start = System.currentTimeMillis();
    f3.join();
    System.out.println("耗时:" + (System.currentTimeMillis() - start) + " ms");
    try {
        //这里阻塞20秒发现f1的任务还会继续执行，只是在结果上先取了f2的数据
        TimeUnit.SECONDS.sleep(20);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

### 场景八：两个任务都执行成功后在消费

两个CompletionStage，都完成了计算才会执行下一步的操作（Runnable）

```java
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

```java
@org.junit.Test
public void testrunAfterBoth(){
    CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "f1";
    });
    CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "f2";
    });
    CompletableFuture<Void> f3 = f1.runAfterBoth(f2, new Runnable() {
        @Override
        public void run() {
            System.out.println("f1,f2都执行成功后执行");
        }
    });
    long start = System.currentTimeMillis();
    f3.join();
    System.out.println("耗时:" + (System.currentTimeMillis() - start) + " ms");
}
```

### 场景九：allOf等待多个任务执行完成

```java
@org.junit.Test
public void testAllOf() throws ExecutionException, InterruptedException {
    ExecutorService executorService = Executors.newCachedThreadPool();
    List<CompletableFuture<String>> futureList = Lists.newArrayList();
    List<String> uuidList = Lists.newArrayList();
    for (int i = 0; i < 10; i++) {
        futureList.add(CompletableFuture.supplyAsync(() -> {
            synchronized (uuidList) {
                for (int j = 0; j < 100; j++) {
                    uuidList.add(UUID.randomUUID().toString());
                }
            }
            System.out.println("threadID:" + Thread.currentThread().getId() + "执行完成");
            return null;
        }, executorService));
    }
    CompletableFuture<Void> result = CompletableFuture.allOf(futureList.toArray(new CompletableFuture[futureList.size()]));
    result.get();
    System.out.println(uuidList.size());
}
```

### 场景十：anyOf 多个任务只要一个任务执行完成

跟acceptEither一样，支持支持多个。

```java
@org.junit.Test
public void testAnyOf() throws ExecutionException, InterruptedException {
    CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
        @Override
        public Integer get() {
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("f1 done");
            return 1;
        }
    });
    CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
        @Override
        public Integer get() {
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("f2 done");
            return 2;
        }
    });
    CompletableFuture<Integer> f3 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
        @Override
        public Integer get() {
            try {
                TimeUnit.SECONDS.sleep(15);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("f3 done");
            return 3;
        }
    });

    CompletableFuture<Object> objectCompletableFuture = CompletableFuture.anyOf(f1, f2, f3);
    long start = System.currentTimeMillis();
    Object o = objectCompletableFuture.get();
    System.out.println("result = " + o + " 消耗时间:" + (System.currentTimeMillis() - start) + " ms");
    TimeUnit.SECONDS.sleep(30);
}
```

## 注意点

### get()和join()的区别

- join()和get()方法都是用来获取CompletableFuture异步之后的返回值
- join()方法抛出的是uncheck异常（即未经检查的异常),不会强制开发者抛出
- get()方法抛出的是经过检查的异常，ExecutionException, InterruptedException 需要用户手动处理（抛出或者 try catch）

### 带Async和不带Async后缀的区别

通常而言，名称中不带 Async 的方法和它的前一个任务一样，在同一个线程中运行。而名称以 Async 结尾的方法会将后续的任务提交到一个线程池，所以每个任务是由不同的线程处理的。

### 带Executor executor和不带的方法如何抉择？

CompletableFuture提供的方法都包括了带<font color=red>Executor executor</font>参数和不带的。那到底怎么选择呢？来看一下不带Executor的方法的底层。如果不传Executor的话，使用的是ForkJoinPool线程池，而ForkJoinPool线程池的大小取决与CPU核数。

```java
private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }
```

- 如果是CPU密集型任务，直接使用ForkJoinPool就可以了
- 如果是IO密集型任务，ForkJoinPool无法达到最佳性能，使用自己传的线程池