---
title: HashMap原理浅析
tags:
  - java集合
categories:  java
description :  HashMap原理浅析
date: 2020-07-28
---
## 基础
### 数组的优点缺点
- 数组在内存空间上连续。能实现快速访问某个下标的值。
- 数组空间的大小一旦确定后就不能更改，如果需要增大减小需要新建数组重新设值。

### 链表的优点缺点

- 链表在内存空间上是不连续的。插入删除某个值速度快
- 链表不能随机查找，必须从第一个开始遍历，查找效率低

### HashMap散列表结构

HashMap采用了散列表结构，结合了数组和链表的优点。

![HashMap结构](hashmap/1.png)

### Hash哈希

Hash也称之为散列。基本原理是把<font color=red>任意长度</font>的输入，通过Hash算法变成<font color=red>固定长度</font>的输出。这个映射的规则就是对应的<font color=red>Hash算法</font>，而原始数据映射后的<font color=red>二进制串</font>就是哈希值。

Hash的特点

- 从hash值不可以反向推导出原始的数据
- 输入数据的微小变化会得到完全不同的hash值，相同的数据会得到相同的值
- 哈希算法的执行效率要高效，长本文也能快速的计算出哈希值
- hash算法的冲突概率要小

由于hash的原理是将输入空间的值映射到hash空间中，而hash值的控件远小于输入的空间。根据抽屉原理，一定会存在不同的输入被映射成相同输出的情况。

## HashMap原理讲解
### HahMap的继承体系

HashMap实现了Map接口，Cloneable接口，Serializable接口

![HashMap继承体系](hashmap/2.png)

### Node数据结构分析

HashMap的Node结构是一个单链表结构。Hash发生碰撞时，此时元素没法放在同一个数组下标，就会将相同hash的Node组成一个单链表。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; //这个hash是通过获取key的hashCode然后经过"扰动"后的结果
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
}
```

### 底层存储结构介绍

底层存储结构是数组+单链表+红黑树的结构。

什么情况下链表会转化成红黑树?<font color=red>当单链表长度达到8，且数组的长度大于64时</font>

![HashMap底层存储结构](hashmap/3.png)

### put数据原理分析
### Hash碰撞
### 什么是链化

### JDK8为什么引入红黑树
### HashMap扩容原理