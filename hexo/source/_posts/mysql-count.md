---
title: MySQL之count(*)、count(1)、count(字段)
tags:
  - mysql
categories:  mysql
description : 详解MySQL之count(*)、count(1)、count(字段)
date: 2020-01-26 08:30:14
---
## 正文
count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加。最后返回累计值。
<!--more-->
### InnoDB下的count

count(*)、count(主键 id) 、 count(1) 都表示返回满足条件的结果集的总行数。而count(字段)则表示返回满足条件的数据行里面，参数“字段”不为 NULL 的总个数。
- **对于count(主键id)来说**，InnoDB引擎会遍历整张表，把每一行的id值取出来，返回给server层。server 层拿到 id 后，判断是不可能为空的，就按行累加。由于主键是不可能为NULL的，所以count(主键id)是忽略NULL的。

  

- **对于 count(1) 来说**，InnoDB 引擎遍历整张表，<font color=red>但不取值</font>。server层对于返回的每一行，放一个数字"1"进去，判断是不可能为空，按行累加。所以count(主键id)是忽略NULL的。

  

- **对于 count(字段) 来说**，所以count(字段)根据字段是否定义为not null来判断是否忽略NULL。
  
  - 如果这个“字段”是定义为 not null 的话，一行行地从记录里面读出这个字段，判断不能为 null，按行累加
  - 如果这个“字段”定义允许为 null，那么执行的时候，判断到有可能是 null，还要把值取出来再判断一下，不是 null 才累加
  
  
  
- **但是 count(\*) 是例外**，并不会把全部字段取出来，而是专门做了优化，不取值。count(*) 肯定不是 null，按行累加。

所以结论是：按照效率排序的话，<font size=5 color=red>count(字段) < count(主键 id) < count(1) ≈ count(\*)</font>，所以我建议你，尽量使用  count(\*)。

### MyISAM下的count

MyISAM 引擎把一个表的总行数存在了磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高。

## 参考

- 《MYSQL实战45讲》