---
title: 记一次MySQL主从同步延迟排查
tags:
  - mysql
categories:  mysql
description : 记一次MySQL主从同步延迟排查和解决
date: 2020-08-19 15:25:29
---

## 问题分析和定位

线上环境这几天在凌晨4点时报主从延迟。虽然我们项目对数据的时效性不是要求很高，但是出现了主从延迟肯定是哪个地方除了问题，为了保证稳定性需要排查一下问题原因。

### 主从同步原理

![](mysql-masterslave-solve/1.jpg)

1. 主库对所有DDL和DML产生的日志写进binlog；
2. 主库生成一个 log dump 线程，用来给从库I/O线程读取binlog；
3. 从库的I/O Thread去请求主库的binlog，并将得到的binlog日志写到relay log文件中；
4. 从库的SQL Thread会读取relay log文件中的日志解析成具体操作，将主库的DDL和DML操作事件重放。

SQL语言共分为四大类：查询语言DQL，控制语言DCL，操纵语言DML，定义语言DD
- DQL：可以简单理解为SELECT语句；
- DCL：GRANT、ROLLBACK和COMMIT一类语句；
- DML：可以理解为CREATE一类的语句；
- DDL：INSERT、UPDATE和DELETE语句都是；

### 主从延迟可能原因
从上面的原理分析，可以知道导致的原因可能是以下几种。
- 网络原因。从库请求主库binlog延迟
- 主库单位时间内产生DDL数量超过从库所能执行的范围
- 从库执行大型的query语句导致锁表，从而导致执行同步过来的DDL语句延迟
- 从库机器性能原因

个人觉得是<font color=red>主库单位时间内产生DDL数量超过从库所能执行的范围</font>这个原因比较大。所以从这开始排查。

## 问题排查

首先登录从库查看

```
mysql -h 从库IP -u 用户名 -p密码 -P 端口
```

```
#执行查看主从同步情况
show slave status\G;
*************************** 1. row ***************************
			  #当前slave I/O状态
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 主库IP
                  Master_User: repl_user
                  Master_Port: 主库端口
                Connect_Retry: 60
              #当前I/O线程正在读取的主服务器二进制日志文件的名称
              Master_Log_File: mysql-bin.000231
          #当前I/O线程正在读取的二进制日志的位置
          Read_Master_Log_Pos: 211281725
          	   #当前slave SQL线程正在读取并执行的relay log的文件名
               Relay_Log_File: mysql-relay-bin.000230
                #当前slave SQL线程正在读取并执行的relay log文件中的位置
                Relay_Log_Pos: 10270447
        Relay_Master_Log_File: mysql-bin.000231
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: wsportal
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: root.domain_info,root.certificate
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          #slave SQL线程当前执行的事件，对应在master相应的二进制日志中的position
          Exec_Master_Log_Pos: 211281725
              Relay_Log_Space: 10270667
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        #slave当前的时间戳和master记录该事件时的时间戳的差值
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 12918051
                  Master_UUID: 8ef54a4c-d0e5-11e7-aa5f-047d7bb918b6
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
```

一般主从同步问题可以通过这几个点排查

- Slave_IO_State 从库当前状态，用来判断从库是否断掉
- Read_Master_Log_Pos 和 Exec_Master_Log_Pos  这个值如果相差太大表示主从出现了延迟
- Seconds_Behind_Master 当前salve和master记录事件的时间戳，这个值如果越大证明延迟越严重

通过看show slave status发现一切正常，因为延迟是发生在凌晨4点。所以有可能的原因就是凌晨4点执行了大量的DDL导致的，之后恢复正常了。所以我们只能通过binlog日志查看凌晨4点时执行了什么语句。

通过上面我们知道了从库是从主库的 mysql-bin.000231日志中同步过来的。所以看下这个binlog日志在凌晨4点时执行的所有DDL语句。

导出主库上凌晨4:00~4:02的binlog日志。

```
#--base64-output=decode-rows -v 表示格式化DDL语句，因为binlog是二进制的
mysqlbinlog --base64-output=decode-rows -v --start-datetime='2020-08-17 04:00:00' --stop-datetime='2020-08-17 04:02:00' -d wsportal mysql-bin.000231 > /home/xuzy/log/mysql_bin_231_4.log
```

```
cat mysql_bin_231_4.log | grep 'INSERT' | wc -l
结果 : 99412
cat mysql_bin_231_4.log | grep 'INSERT' | grep 'dir_visit_report' | wc -l
结果 : 43695
```

通过查看binlog日志，发现在2分钟内主库插入数据到表dir_visit_report一共执行了4万多次。感觉原因就是这个，然后继续查了下业务代码，发现是同事一个入库定时器写错了写成每天凌晨4点执行，其实是一个月执行一次，所以导致每天都报延迟。

## 如何避免

待续。。。