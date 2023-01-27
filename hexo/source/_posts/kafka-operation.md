---
title: Kafka运维笔记（二）
tags:
  - kafka
categories:  kafka
description : Kafka运维笔记
date: 2019-11-16 09:22:03
---

<!--more-->

### 常用命令

**创建主题**

```shell
#window
.\bin\windows\kafka-topics.bat --create  --zookeeper localhost:2181 --replication-factor 1 --partitions 6 --topic test001
#shell
sh kafka-topics.sh --create  --zookeeper localhost:2181 --replication-factor 1 --partitions 6 --topic test001
```

### 配置优化
待续。。。
