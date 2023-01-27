---
title: 设计模式之单例模式
tags:
  - 设计模式
categories:  java
description : 设计模式之单例模式
date: 2019-07-18 10:00:00
---
<!--more-->

## 单例模式

### 概念
单例模式(Singleton Pattern)：确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例，这个类称为单例类，它提供全局访问的方法。单例模式是一种对象创建型模式
### 特性
单例模式有三个特性：
- 单例类只能有一个实例
- 单例类必须自行创建自己的唯一的实例
- 单例类必须给所有其他对象提供这一实例

### 单例模式的几种实现
#### 懒汉式单例模式
```java
public class SingletonOne {
    private static SingletonOne singletonOne;
    private SingletonOne(){}
    private static SingletonOne getInstance(){
        if(singletonOne == null)
            singletonOne = new SingletonOne();
        return singletonOne;
    }
}
public static void main(String[] args){
    SingletonOne singletonOne = SingletonOne.getInstance();
}
缺陷 ：
```
- 不支持多线程，在并发环境下会导致创建多个实例

#### 懒汉式线程安全单例模式

```java
public class SingletonTwo {
    private static SingletonTwo singletonTwo;
    private SingletonTwo(){}
    private synchronized static SingletonTwo getInstance(){
        if(singletonTwo == null)
            singletonTwo = new SingletonTwo();
        return singletonTwo;
    }
}
public static void main(String[] args){
    SingletonTwo singletonOne = SingletonTwo.getInstance();
}
```

这种方式是在懒汉式单例模式下加了synchronized关键字保证线程安全

缺陷：

- 虽然能保证线程安全，但是用了synchronized，加锁影响效率

#### 饿汉式单例模式

```java
public class SingletonThree {
    private static SingletonThree singletonThree = new SingletonThree();
    private SingletonThree(){}

    private static SingletonThree getInstance(){
        return singletonThree;
    }
}
public static void main(String[] args){
    SingletonThree singletonOne = SingletonThree.getInstance();
}
```

特点与缺陷：

- 没有加锁，执行效率高
- 类加载时就初始化，浪费内存

#### 双检查锁单例模式

```java
public class SingletonFour {
    private static volatile SingletonFour singletonFour;
    private SingletonFour(){}
    private static SingletonFour getSingleton(){
        if(singletonFour == null){  //第一次检查
            synchronized (SingletonFour.class){
                if(singletonFour == null) //第二次检查
                    singletonFour = new SingletonFour(); // 标注1
            }
        }
        return singletonFour;
    }
}
public static void main(String[] args){
    SingletonFour singletonOne = SingletonFour.getInstance();
}
```

这种方式采用了二次检查+ synchronized的方式，且采用了volatile修饰单例。第一次检查可以保证大部分线程不进入synchronized判断，第二次检查保证了少部分进入synchronized的线程不重新new 对象。

**那为什么要用volatile修饰？**

因为singletonFour = new SingletonFour() 包含了如下三步操作。

```java
memory = allocate();　　// 1.分配对象的内存空间
ctorInstance(memory);　　// 2.初始化对象
sInstance = memory;　　// 3.设置sInstance指向刚分配的内存地址
```

上述伪代码中的2和3之间可能会发生重排序，重排序后的执行顺序如下

```java
memory = allocate();　　// 1.分配对象的内存空间
sInstance = memory;　　// 2.设置sInstance指向刚分配的内存地址，此时对象还没有被初始化
ctorInstance(memory);　　// 3.初始化对象
```

所以在多线程下，如果发生重排序，singletonFour == null会返回ture，导致进行了两次初始化对象。

#### 静态内部类单例模式

```java
public class SingletonFive {
    private SingletonFive(){}
    //创建一个内部静态类
    private static class SingletonHolder{
        private final static SingletonFive sinleton = new SingletonFive();
    }
    public static SingletonFive getSingleton(){
        return SingletonHolder.sinleton;
    }
}
public static void main(String[] args){
    SingletonFive singletonOne = SingletonFive.getInstance();
}
```

我们在单例类中增加一个静态(static)内部类，在该内部类中创建单例对象，再将该单例对象通过getInstance()方法返回给外部使用。由于静态单例对象没有作为Singleton的成员变量直接实例化，因此类加载时不会实例化Singleton，第一次调用getInstance()时将加载内部类SingletonHolder，在该内部类中定义了一个static类型的变量instance，此时会首先初始化这个成员变量，由Java虚拟机来保证其线程安全性，确保该成员变量只能初始化一次。由于getInstance()方法没有任何线程锁定，因此其性能不会造成任何影响。通过使用IoDH，我们既可以实现延迟加载，又可以保证线程安全，不影响系统性能，不失为一种最好的Java语言单例模式实现方式（其缺点是与编程语言本身的特性相关，很多面向对象语言不支持IoDH）

## 参考

- http://tengj.top/2016/04/06/sjms4singleton/

