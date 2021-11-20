---
title: influxdb学习(一)
tags:
  - influxdb
categories:  influxdb
description : influxdb学习(一)
date: 2021-02-10 09:05:24
---

### 核心概念

```
> insert cpu_usage,host=server01,location=cn-sz user=23.0,system=57.0
> select * from cpu_usage
name: cpu_usage
time             host     location system user
----             ----     -------- ------ ----
1557834774258860710 server01 cn-sz    55     25
```

- **时间（Time）：**如代码清单1-3中的“1557834774258860710”，表示数据生成时的时间戳，与MySQL不同的是，在InfluxDB中，时间几乎可以看作主键的代名词。
- **表（Measurement）：**如代码清单1-3中的“cpu_usage”，表示一组有关联的时序数据，类似于MySQL中表（Table）的概念。
- **标签（Tag）：**如代码清单1-3中的“host=server01”和“location=cn-sz”，用于创建索引，提升查询性能，一般存放的是标示数据点来源的属性信息，在代码清单1-3中，host和location分别是表中的两个标签键，对应的标签值分别为server01和cn-sz。
- **指标（Field）：**如代码清单1-3中的“user=23.0”和“system=57.0”，一般存放的是具体的时序数据，即随着时间戳的变化而变化的数据，与标签不同的是，未对指标数据创建索引，在代码清单1-3中，user和system分别是表中的两个指标键，对应的指标值分别为23.0和57.0。
- **时序数据记录（Point）：**如代码清单1-3中的“1557834774258860710 server01 cn-sz 55 25”，表示一条具体的时序数据记录，由时序（Series）和时间戳（Timestamp）唯一标识，类似于MySQL中的一行记录。 
- **保留策略（Retention Policy）：**定义InfluxDB的数据保留时长和数据存储的副本数，通过设置合理的保存时间（Duration） 和副本数（Replication），在提升数据存储可用性的同时，避免数据爆炸。
- **时间序列线（Series）：**表示表名、保留策略、标签集都相同的一组数据。