---
title: java并发之中断详解
tags:
  - java并发
categories:  java
description : 详解java中断线程的几种方式和方法interrupt()，Thread.interrupt()，isInterrupted()区别
date: 2020-08-27 10:57:08
---

## 概述

java中提供了多种方式来实现线程的中断。本文通过列举其中的方式。并对方法interrupt()，Thread.interrupt()，isInterrupted()进行区分。
<!--more-->
## 中断方法分析

涉及中断的方法有如下三个，

| 方法                 | 方法类型             | 说明                                                         |
| -------------------- | -------------------- | ------------------------------------------------------------ |
| interrupt()          | Thread类下的实例方法 | 作用是中断线程，只是改变中断状态不会中断线程，后续就是根据这个中断状态去调用isInterrupted()或Thread.interrupted()或抛出interruptedException异常来结束中断 |
| isInterrupted()      | Thread类下的实例方法 | 测试线程是否已经中断，不会重置当前线程的中断状态             |
| Thread.interrupted() | Thread类下的静态方法 | 测试当前线程(内部用的是currentThread())是否已经中断，会重置当前线程的中断状态 |

说明：

- 会不会重置当前线程的中断状态指的是如果原来是中断，调用Thread.interrupted()就重新设置成不中断了
- 两个判断中断的方法最终都是调用本地方法boolean isInterrupted(boolean ClearInterrupted)

```java
//Thread.interrupted();
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}
//this.isInterrupted
public boolean isInterrupted() {
    return isInterrupted(false);
}
private native boolean isInterrupted(boolean ClearInterrupted);
```



## stop方法中断

<font color=red>不推荐使用</font>。使用stop方法虽然可以`强行终止`正在运行或挂起的线程，但使用stop方法是很`危险`的，就象突然关闭计算机电源，而不是按正常程序关机一样，可能会产生不可预料的结果，因此，并不推荐使用stop方法来终止线程。

## 代码上使用标志位中断

在线程执行方法run中使用一个循环体，如果想使while循环在某一特定条件下退出，最直接的方法就是设一个boolean类型的标志，并通过设置这个标志为true或false来控制while循环是否退出。

```java
public class myTest {
	//这里必须使用volatile修饰保证可见性
    public static volatile boolean exit =false;  //退出标志
    public static void main(String[] args) {
        new Thread() {
            public void run() {
                System.out.println("线程启动了");
                while (!exit) {
                    System.out.println("模拟运行很长的代码段............");
                }
                System.out.println("线程结束了");
            }
        }.start();
        
        try {
            Thread.sleep(1000 * 5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        exit = true;//5秒后更改退出标志的值,没有这段代码，线程就一直不能停止
    }
}
```

## interrupt()+InterruptedException中断线程

线程处于阻塞状态，如Thread.sleep、wait、IO阻塞等情况时，调用interrupt方法后，sleep等方法将会抛出一个InterruptedException。通过异常退出的方式

```java
public class Interrupt2 {
    public static void main(String[] args) {
        Thread thread = new Thread(new MyThread(), "testThread");
        thread.start();
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {

        }
        MyThread.on = true;
        thread.interrupt();
    }

    static class MyThread extends Thread {
        private volatile static boolean on = false;
        @Override
        public void run() {
            //通过抛出异常结束阻塞中的代码块，然后通过标志判断退出循环
            //好像中断都需要写在循环里比较安全
            while (!on) {
                try {
                    System.out.println("begin Sleep");
                    Thread.sleep(10000000);
                    System.out.println("end Sleep");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("Oh, myGod!");
            }
        }
    }
}
```

## interrupt()+isInterrupted()中断线程

通过设置中断状态+判断中断标志来中断线程。

```java
public class Interrupt {
    public static void main(String[] agrs) {
        try{
            MyThread myThread = new MyThread();
            myThread.start();
            TimeUnit.SECONDS.sleep(1); //休眠两秒
            myThread.interrupt(); //请求中断
        }catch (Exception e){
            System.out.println("main catch");
            e.printStackTrace();
        }
    }

    static class MyThread extends Thread {
        @Override
        public void run() {
            for (int i = 0; i < 500000; i++) {
                if (this.isInterrupted()) {
                    System.out.println("should be stopped and exit");
                    break;
                }
                System.out.println("i=" + (i + 1));
            }
            System.out.println("this line is also executed. thread does not stopped"); //尽管线程被中断,但并没有结束运行。这行代码还是会被执行
        }
    }
}
```

