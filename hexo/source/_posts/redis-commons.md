---
title: Redis常用数据结构命令
tags:
  - redis
categories: redis
description : 主要用来记录Redis常用命令的记录，方便查询
date: 2019-10-03 08:32:58
---

### key常用操作

```properties
# 查询指定key是否存在
$ localhost:6379> EXISTS hash1
(integer) 1
$ localhost:6379> EXISTS jjj
(integer) 0

# 删除指定key
$ localhost:6379> DEL hash1
(integer) 1
$ localhost:6379> EXISTS hash1
(integer) 0

# 对指定key设置过期时间
$ localhost:6379> set t1 value ex 5
OK

# 查询指定key的过期时间
$ localhost:6379> TTL t1
(integer) 7

# 查询指定key的类型
$ localhost:6379> type t1
string
```
<!--more-->
### kv结构

```properties
# 设置key 对应的value
$ localhost:6379> set key1 value1
OK

# 获取key对应的value
$ localhost:6379> get key1
"value1"

# Redis Setex 命令为指定的 key 设置值及其过期时间。如果 key 已经存在， SETEX 命令将会替换旧的值。
$ localhost:6379> SETEX mykey 60 redis
OK

# 命令在指定的 key 不存在时，为 key 设置指定的值。
$ localhost:6379> setnx key3 vlaue3
OK
$ localhost:6379> setnx key3 value3
(nil)
```

### list结构

```properties
# 从左侧插入元素
$ localhost:6379> LPUSH list1 1
(integer) 1

# 从右侧插入元素
$ localhost:6379> RPUSH list1 2
(integer) 2

# 查看集合内的元素，-1 表示查看所有元素
$ localhost:6379> LRANGE list1 0 -1
1) "1"
2) "2"

# 查看list的元素个数
$ localhost:6379> LLEN list1
(integer) 2

# 根据索引查询对应的元素，如果指定的索引不存在，则返回'nil'
$ localhost:6379> LINDEX list1 0
"2"

# 从列表左侧移除一个元素
$ 127.0.0.1:6379> LPOP list1
"5"

# 从列表右侧移除一个元素
$ 127.0.0.1:6379> RPOP list1
"1"

# 从列表右侧移除一个元素添加到左侧
$ localhost:6379> LRANGE list1 0 -1
1) "three"
2) "two"
3) "one"
$ localhost:6379> RPOPLPUSH list1 list2
"one"
$ localhost:6379> LRANGE list2 0 -1
1) "one"
```

### set结构

```properties
# 向set中添加一个元素
$ localhost:6379> SADD set1 'one' 'two' 'three'
(integer) 3

# 获取set集合中的元素
$ localhost:6379> SMEMBERS set1
1) "one"
2) "three"
3) "two

# 从set集合中移除一个或多个元素
$ localhost:6379> SREM set1 'one'
(integer) 1

# 从set集合中移除一个或多个元素并返回被删除元素
$ localhost:6379> SPOP set1 1
1) "three"

# 获取当前set集合元素个数
$ localhost:6379> SCARD set1
(integer) 3

# 从set集合随机获取元素但不删除
$ localhost:6379> SRANDMEMBER set1 1
1) "one"
$ localhost:6379> SMEMBERS set1
1) "three"
2) "two"
3) "one"

# 判断set集合中是否存在指定元素，如果存在则返回1，不存在返回0
$ localhost:6379> SISMEMBER set1 'one'
(integer) 1
$ localhost:6379> SISMEMBER set1 '4'
(integer) 0
```

### sorted set  结构

```properties
# 向有序集合中添加元素
$ localhost:6379> ZADD zset1 1 'one'
(integer) 1

# 获取有序集合中指定分数范围的元素
$ localhost:6379> ZRANGE zset1 0 -1
1) "one"
2) "two"
3) "three"

# 删除有序集合中的元素
$ localhost:6379> ZREM zset1 'one'
(integer) 1

# 获取有序集合元素个数
$ localhost:6379> ZCARD zset1
(integer) 2

# 为有序集合中指定成员增加指定个数
$ localhost:6379> ZINCRBY zset1 2 "one"
"4"
$ localhost:6379> ZRANGE zset1 0 -1 WITHSCORES
1) "two"
2) "2"
3) "three"
4) "3"
5) "one"
6) "4"

# 获取有序集合指定分数范围内的元素数量
$ localhost:6379> ZCOUNT zset1 0 2
(integer) 1

# 获取元素在有序集合中的排名，分数越大，排名值越大
$ localhost:6379> ZRANK zset1 'one'
(integer) 2

# 获取元素在有序集合中的排名。分数越大，排名值越小
$ localhost:6379> ZREVRANK zset1 'one'
(integer) 0

# 获取指定元素的分值
$ localhost:6379> ZSCORE zset1 'one'
"4"
```

### hash结构

```properties
# 向hash集合中添加一个元素
$ localhost:6379> HSET hash1 field1 1
(integer) 1

# 向hash集合中添加多个元素
$ localhost:6379> HMSET hash1 field2 2 field3 3
OK

# 获取指定field对应的value
$ localhost:6379> HGET hash1 field1
"1"

# 批量获取指定field下的value
$ localhost:6379> HMGET hash1 field1 field2 field3
1) "1"
2) "2"
3) "3"

# 获取hash结合里面所有元素
$ localhost:6379> HGETALL hash1
1) "filed1"
2) "1"
3) "filed2"
4) "2"
5) "filed3"
6) "3"
7) "field3"
8) "3"

# 判断指定filed是否在Hash结构中存在
$ localhost:6379> HEXISTS hash1 field1
(integer) 1
$ localhost:6379> HEXISTS hash1 field5
(integer) 0

# 从hash结构中删除一个或多个field
$ localhost:6379> HDEL hash1 field1 field2
(integer) 2

# 对hash集合中指定field的value增加值
$ localhost:6379> HINCRBY hash1 field1 2
(integer) 3
$ localhost:6379> HGET hash1 field1
"3"

# 获取hash结构中所有的key
$ localhost:6379> HKEYS hash1
1) "filed1"
2) "filed2"
3) "filed3"
4) "field3"
5) "field1"

$ 获取hash集合中所有的value
$ localhost:6379> HVALS hash1
1) "1"
2) "2"
3) "3"
4) "3"
5) "3"

# 获取hash集合中的元素个数
$ localhost:6379> HLEN hash1
(integer) 5
```

### 总结

具体命令地址 : http://redis.cn/commands.html