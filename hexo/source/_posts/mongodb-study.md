---
title: MongoDb学习
tags:
  - MongoDb
categories: MongoDb
description : MongoDb学习笔记
date: 2022-01-01 15:46:34
---
<!--more-->

## MongoDB与传统数据库的概念

| 传统     |        | MongoDB    |                         |
| -------- | ------ | ---------- | ----------------------- |
| 概念     | 说明   | 概念       | 说明                    |
| database | 数据库 | database   | 数据库                  |
| table    | 表     | collection | 集合                    |
| row      | 行     | document   | 文档                    |
| columm   | 列     | filed      | 字段: BJSON具体某个字段 |

![](mongodb-study\1.png)

![](mongodb-study\2.png)

## MongoDB入门命令

```sql
# 使用testdb数据库
use testdb

# 创建集合 集合规则可以省略
# capped :选填bool类型：设置改集合是否为一个固定集合true:代表固定集合，集合中的数据不可修改，与size配对使用，代表当集合达到指定大小后，会自动覆盖历史数据（最先添加的数据）
# size:选填数字类型：指定集合的最大存储数据（字节数），当集合达到指定大小后，会自动覆盖历史数据（最先添加的数据） }
# max: 选填数字类型：指定集合的最大存储的文档总个数，当文档个数大于max值时，会自动替换历史文档
db.createCollection("集合名称", {集合规则});
db.createCollection("user001", {capped : true, size : 10000000, max});

# 集合删除
db.集合名称.drop();

# 插入数据
db.集合名称.insert(json对象|json对象数组);
db.user001.insert({name: "xzy", age : 10});
db.user001.insert([{name: "xzy01", age : 10},{name: "xzy01", age : 20}]);

# 更新数据
# query ：被更新文档条件json
# update：更新后的文档json
# option：更新方式json，参数格式为{ upsert: boolean, multi : boolean }
# upsert:非必填参数，如果不存在是否新增，当值为true时，如果没有符合条件的数据，就插入数据, ，默认为false
# multi: 非必填参数，是否更新符合要求的所有数据，当值为true时，符合条件的数据全部更新，默认为false
db.集合名称.update(query, update, option);

# 删除数据
# query :（可选）删除的文档的条件。
# justOne : （可选）如果设为 true 或 1，则只删除一个文档，如果不设置该参数，或使用默认值 false，则删除所有匹配条件的文档。
db.集合名称.remove(query, justOne);

# 查询数据
db.集合名称.find(json对象查询条件)

```

## MongoDB数据类型

![](mongodb-study\3.png)

## MongoDB查询详解

查询的语句格式 : `db.集合名称.find(query, projection)`

- query : 是一个查询条件BJSON对象，根据查询条件构建对应的BJSON对象
- projection : 设置查询需要返回的字段集合，不设置代表返回全部字段，其格式为:{字段名称:是否获取}，当设置为1代表需要获取，注意：_id默认值为1，所以需要查询结果不需要_id，那么需要设置其值为0

```
use testdb
# 初始化3条数据
db.user001.insert([
	{name:"java",age:2,from: "CTU"},
	{name: "mongdo",age:13,from: "USA"},
	{name: "C++",from: "USA" ,author:["xiaoxu","xiaozhang"]}
]);

# 查询所有数据
db.user001.find()
# 查询name=java的数据，并且只返回name和from字段
db.user001.find({name : "java"}, {_id : 0, name : 1, from : 1})
```

高级查询 :  `db.集合名称.find({字段: { 查询符 : 值} })`

mongodb的单个查询符包括 : 

- **比较符：**（等于**[****严格来说等于不是一个查询符****]**、不等于(ne)、大于(gt)、小于(lt)、大于等于(gte)、小于等于(lte)）
- **包含符：**（包含(in)、不包含(nin)、全包含(all)）；**模糊匹配符**（左匹配(/^X/)、右匹配(/X$/)、模糊匹配(/X/ )、全部匹配(/^X$/)）
- **元素操作符：**（字段是否存在(exists)、字段值是否为空(null)、字段类型(type)）
- **集合查询符：**（长度、子查询）
- **取模符：**实际上就是取余数
- **正则表达式：**js正则表达式

```
use testdb
db.user001.insert({name:".net",age:88,author: ["xiaoxu","xiaozhang","xiaoliu"]});

# 查询from != 'USA'的数据
db.user001.find({ from : {$ne: "USA"} })

# 查询 age > 13 的数据
db.user001.find({age: {$gt: 13}})

# 查询from在["CTU","USA"]的数据
db.user001.find(from : {$in: ["CTU","USA"]})

# 查询 from不在["CTU","USA"]的数据
db.user001.find(from : {$nin: ["CTU","USA"]})

# 全包含和包含的区别，全包含更精准，需要满足多个
# 查询author全部包含["xiaoxu","xiaozhang"] 的数据
db.user001.find({author:{$all:["xiaoxu","xiaozhang"] }})

# 查询包含author的集合数据, 只要这一列有这个字段就可以查出来
db.user001.find({author: {$exists: true}})

# 查询不包含author的集合数据
db.user001.find({author:{$exists :false}})
```

mongodb的终极查询

- 逻辑查询符 : 逻辑操作符其实简单的理解就是将不同的单元查询符组合，通过逻辑运算符来进行逻辑判断。逻辑查询符主要包括：$and、$or、$nor、$not。
- 排序 : Mongodb排序实现上很简单通过sort()方法，指定参数来排序，并可以根据一个或者多个节点来排序。语法 `.sort({file1: sortType,...,filen: sortType})`
- 分页查询 : 分页查询其效果就是要实现从某一个位置开始取指定条数的数据。这就引出了两个方法，查找开始(skip)，获取指定条数数据(limit)
  - skip语法为skip（num）:指跳过指定条数（num）的数据
  - limit语法为limit（num）:指限制只获取num条数据



## 参考

- http://www.xyhkj.top/mongodb/mongodb.html