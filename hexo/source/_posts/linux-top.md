---
title: Linux故障排查命令使用(top、iostat、vmstat、netstat)
tags:
  - linux
categories:  linux
description: Linux故障排查命令使用
---
## top命令详解
### 各项指标详解
![top命令展示](linux-top/1.png)

#### 第一行：系统运行时间和平均负载
```shell
top - 14:46:47 up 909 days,  3:15,  1 user,  load average: 1.80, 1.50, 1.37
```
- 14:46:47 表示当前系统时间
- up 909 days 表示已经运行了909天期间没重启
- 1 user 表示当前登录的用户数量
- load average: 1.80, 1.50, 1.37 表示当前系统5分钟;10分钟;15分钟的负载
<font color=red>load average数据是每隔5秒钟检查一次活跃的进程数，然后按特定算法计算出的数值。如果这个数除以逻辑CPU的数量，结果高于5的时候就表明系统在超负荷运转了。</font>

#### 第二行：任务信息
```shell
Tasks: 652 total,   1 running, 651 sleeping,   0 stopped,   0 zombie
```
- total : 当前系统的任务进程个数
- runnning : 正在运行的任务进程个数
- sleeping : 正在休眠的任务进程个数
- stopped : 已经终止的任务进程个数
- zombie : 僵尸进程个数

running 表示在运行，占用cpu，sleeping则多半是在等待（比如等IO完成），所以大部分进程在sleeping是很正常的，否则都在running，cpu都不够用了.

#### 第三行：CPU信息
```shell
Cpu(s):  3.8%us,  0.4%sy,  0.0%ni, 95.7%id,  0.1%wa,  0.0%hi,  0.0%si,  0.0%st
```
- us — 用户空间占用CPU的百分比。
- sy — 内核空间占用CPU的百分比。
- ni — 改变过优先级的进程占用CPU的百分比
- id — 空闲CPU百分比
- wa — IO等待占用CPU的百分比
- hi — 硬中断（Hardware IRQ）占用CPU的百分比
- si — 软中断（Software Interrupts）占用CPU的百分比

这里的内核态和用户态可以这样理解：
- 当一个任务（进程）执行系统调用而陷入内核代码中执行时，称进程处于内核运行态（内核态）。
- 当进程在执行用户自己的代码时，则称其处于用户运行态（用户态）。

#### 第四五行：内存信息
```shell
Mem:  231349636k total, 72512612k used, 158837024k free,   299360k buffers
Swap:  8388600k total,        0k used,  8388600k free,  3648000k cached
```
- Mem
 - total 物理内存重量
 - used 使用中的内存总量
 - free 空间内存总量
 - buffers 缓存的内存量
- swap交换分区 具体值与上面的Mem一样,只是是对应交换分区的

#### 各任务状态监控
```
   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
   10840 tomcat  20   0 68.0g  21g  14m S  0.3  9.8  71:32.92 jsvc     
```
- PID：进程ID，进程的唯一标识符
- USER：进程所有者的实际用户名。
- PR：进程的调度优先级。这个字段的一些值是'rt'。这意味这这些进程运行在实时态。
- NI：进程的nice值（优先级）。越小的值意味着越高的优先级。负值表示高优先级，正值表示低优先级
- VIRT：进程使用的虚拟内存。进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
- RES：驻留内存大小。驻留内存是任务使用的非交换物理内存大小。进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
- SHR：SHR是进程使用的共享内存。共享内存大小，单位kb
- S：这个是进程的状态。它有以下不同的值:
 - D - 不可中断的睡眠态。
 - R – 运行态
 - S – 睡眠态
 - T – 被跟踪或已停止
 - Z – 僵尸态
- %CPU：自从上一次更新时到现在任务所使用的CPU时间百分比。
- %MEM：进程使用的可用物理内存百分比。
- TIME+：任务启动后到现在所使用的全部CPU时间，精确到百分之一秒。
- COMMAND：运行进程所使用的命令。进程名称（命令名/命令行）

### top 参数命令
- top -p PID 只监控某一个进程
- top -c c表示展示完整命令行
- top -Hp PID 查看指定进程的线程数的信息 <font>这个通常可以配合jstack来判断哪个线程占用CPU或者内存多</font>

### top 交互命令
在top视图下，我们可以键入一些命令在帮助我们获得不同的视图

#### 键盘数字1
在top视图下，按数字1可以展示每个CPU的使用情况

![](linux-top/2.png)

#### 常用键盘
- P ：根据CPU使用百分比大小进行排序
- M ：根据驻留内存大小进行排序
- c ：切换显示命令名称和完整命令行
- T ：根据时间/累计时间进行排序
- q ：退出程序

## 参考
- https://www.cnblogs.com/Diyo/p/11411157.html
- https://mp.weixin.qq.com/s/Vw63MUA0Zt80cU8_mvu7QQ