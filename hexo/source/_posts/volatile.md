---
title: java并发之volatile
tags:
  - java
  - java并发
categories:  java
description : 详解volatile用法和原理
date: 2019-07-18 10:00:00
---

## Java内存模型(jmm)
看个例子
```java
public class VolatileExample extends Thread{
    //设置类静态变量,各线程访问这同一共享变量
    private  static boolean flag = false;
    //无限循环,等待flag变为true时才跳出循环
   public void run() {
       while (!flag){
       };
       System.out.println("停止了");
   }

    public static void main(String[] args) throws Exception {
        new VolatileExample().start();
        //sleep的目的是等待线程启动完毕,也就是说进入run的无限循环体了
        Thread.sleep(100);
        flag = true;
    }
}
```
上面程序运行后发现当主线程设置flag为true时子线程的循环并没有停掉。

***为什么会出现上面的问题?***

Java内存模型规定所有的变量都是存在主存当中（类似于前面说的物理内存），每个线程都有自己的工作内存（类似于前面的高速缓存）。线程对变量的所有操作都必须在工作内存中进行，而不能直接对主存进行操作。并且每个线程不能访问其他线程的工作内存。如下图

![](volatile/1.png)
所以根据JMM规范，每个线程都有单独的一个工作内存，当主线程在运行时会将flag变量的值拷贝一份到自己的工作内存中，当改变了flag时但是还没来得及写入主存中，主线程转去做其他事情，那么子线程就不知道flag的更改，所以就一直循环下去。

## volatile详解
### 线程中的三个概念
- 原子性，对基本数据类型的变量的读取和赋值操作是原子性操作，即这些操作是不可被中断的，要么执行，要么不执行
- 可见性，保证修改的值会立即被更新到主存，当有其他线程需要读取时，它会去内存中读取新值
- 有序性，在Java内存模型中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行
### volatile作用
一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：
- 保证了不同线程对volatile修饰的变量进行改动时，其他线程能马上可见
- 禁止执行重排序
#### volatile如何保证可见性
待续 -- volatile的编译器原理
#### volatile 有序性
待续 --
#### volatile 不支持原子性
待续 --

## java中使用volatile
待续。。。

## 总结
- volatile保证了线程的可见性，当一个线程对其修饰volatile的变量进行改动时会立即被更新到主存