---
title: Redis数据结构总结
tags:
  - redis
categories: redis
description : Redis数据结构总结
date: 2019-10-03 08:32:58
---
### 概述

本文主要总结Redis几种数据结构的特点，Redis由于单线程和纯内存的特点，所以在数据结构上也做了很多优化，了解Redis的数据结构有利于我们更好的根据实际场景使用正确的数据结构。

### 字典

字典是Redis中用来存储Key-Value的数据结构。Redis数据库使用字典来作为底层的实现，对数据库的增删改查也是构建在字典上。

```shell
redis>set name "xuzy"
OK
```

除了用于表示数据库之外，字典也是哈希键的底层实现。

```shell
redis >hlen website
(integer) 10086
redis >hgetall website
1)"xuzy"  //key
2)"chen"  //value
...
```

字典使用哈希表作为底层实现。数据结构如下

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    //哈希表
    dictht ht[2];
    int trehashidx;
}
typedef struct dictht {
    //哈希表数组
	dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值 总是等于size - 1
    unsigned long sizemask;
    //哈希表已有节点的数量
    unsigned long used;
} dictht;

//哈希节点
typedef struct dictEntry{
    void *key;
    union {
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    //指向下一个节点
    struct dictEntry *next;
}

```

![](redis-data-type/1.png)

- 字典采用哈希表的结构。初始化大小为4的哈希数组。使用<font color=red>链地址</font>法来解决冲突，被分配到同一个索引的多个键值会连接成一个单向链表。
- 每个字典带有两个哈希表（ht[0]、ht[1]）。一个平时使用，一个在进行rehash使用。
- Redis字典的rehash并不是一次性完成的，试想一下一个哈希表中有上百万个键值，如果一次性rehash会拖慢Redis的速度，所以是**渐进式**完成的。

### 整数集合

整数集合用于保存**整数值**的集合抽象数据结构，并且保证**集合中不会出现重复元素**。

  

```c
typedef struct intset{
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
}
```

未完待续。。。