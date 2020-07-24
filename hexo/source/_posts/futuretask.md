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
future在字面上表示未来的意思，在Java中一般通过继承Thread类或者实现Runnable接口这两种方式来创建多线程，但是这两种方式都有个缺陷，就是不能在执行完成后获取执行的结果。然而JDK提供了一种类似ajax的方式，允许提交任务后去做自己的事，在任务执行完成后可以获得执行的结果。
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