---
title: netty学习笔记
date: 2023-01-27 20:24:38
tags: 
	- netty
categories: netty
---

## Netty进阶之路读书笔记

### Netty服务端意外退出问题

下面代码执行后进程就结束了

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
//..省略代码
ChannelFuture f = serverBootstrap.bind(8080).sync();
```

因为`serverBootstrap.bind(8080)`的作用是`异步`执行服务端启动，`sync()`表示`同步`等待服务端启动完成。由于服务端启动速度很快，所以执行完后若没有其他代码阻塞住main方法，则main方法执行完进程就退出了。

改造

```java
ServerBootstrap serverBootstrap = new ServerBootstrap();
//..省略代码
ChannelFuture f = serverBootstrap.bind(8080).sync();
f.channel().closeFuture().addListener((ChannelFutureListener) channelFuture -> {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();	 
}).sync();
```

`closeFuture()`表示异步阻塞等待服务端关闭，`sync()`表示同步等待阻塞结果。所以加了上面的代码就能在main方法上阻塞住，保证服务端一直启动。同时在关闭的时候加了一个回调函数对netty资源进行释放。

### Netty客户端连接池资源泄露

如果netty需要建立多个client，代码上该怎么写?

### Netty性能调优

#### 操作系统参数调优

```shell
vi /etc/sysctl.conf
fs.file-max = 1000000 # 设置系统最大文件句柄

vi /etc/security/limits.conf
# 当并发接入的TCP连接数超过上限时会出现"too many open files",这里设置进程打开的文件描述符数量
soft nofile 1000000
hard nofile 1000000

vi /etc/sysctl.conf
#表示本机作为客户端对外发起tcp/udp连接时所能使用的临时端口范围
net.ipv4.ip_local_port_range = 1024 65535
# 内核分配给TCP连接的内存
net.ipv4.tcp_mem = 786432 2097152 3145728
#TCP接收缓冲区大小(min，default，max)
net.ipv4.tcp_rmem = 4096 4096 16777216
#TCP发送缓冲区大小(min，default，max)
net.ipv4.tcp_wmem = 4096 4096 16777216
#开启端口复用 被 TIME_WAIT 状态占用的端口，还能用到新建的连接中
net.ipv4.tcp_tw_reuse = 1
#开启TCP连接中TIME-WAIT sockets的快速回收
net.ipv4.tcp_tw_recycle = 1
```

#### Netty参数调优

- 设置合理的线程数，一般默认的不该够用了，如果不够用就更改`NioEventLoopGroup`的参数。
- 心跳优化，使用`IdleStateHandler`时不要把时间设置过长，因为他底层会设置一个定时器来检测是否有读写请求，若时间设置过长当大量请求接入时内存会维护多个定时器导致内存暴涨。
- 接收和发送缓冲区调优，主要是`ChannelOption.SO_RCVBUF`和`ChannelOption.SO_SNDBUF`。
- IO线程和业务线程分离，对于耗时高的方法最好使用单独的线程池去处理，如果耗时高的方法在`NioEventLoopGroup`上处理会占用IO线程池的资源。
- 防止IO线程被意外阻塞，