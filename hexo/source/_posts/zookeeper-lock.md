---
title: Zookeeper实现分布式锁（附代码）
tags:
  - zookeeper
categories:  zookeeper
description : 详解Zookeeper实现分布式锁
date: 2020-08-27 17:06:06
---




## 什么是分布式锁

在日常开发中，我们最熟悉也常用的分布式锁场景是在开发多线程的时候。为了协调本地应用上多个线程对某一资源的访问，就要对该资源或数值变量进行加锁，以保证在多线程环境下系统能够正确地运行。在一台服务器上的程序内部，线程可以通过系统进行线程之间的通信，实现加锁等操作。而在分布式环境下，执行事务的线程存在于不同的网络服务器中，要想实现在分布式网络下的线程协同操作，就要用到分布式锁。

## 独占式非公平分布式锁

### 概述

独占式非公平锁采用的方式是通过创建节点，如果节点创建成功则表示加锁成功，如果创建不成功则需要等待并对节点(锁)设置watch。

当节点(锁)释放后会触发watch的节点删除事件，从而重新抢占创建节点。

![](zookeeper-lock/1.png)

1. 多个客户端竞争创建 lock 临时节点
2. 其中某个客户端成功创建 lock 节点，其他客户端对 lock 节点设置 watcher
3. 持有锁的客户端删除 lock 节点或该客户端崩溃，由 Zookeeper 删除 lock 节点
4. 其他客户端获得 lock 节点被删除的通知
5. 重复上述4个步骤，直至无客户端在等待获取锁了

### 代码实例

**主代码**

```java
package lock;

import org.apache.commons.lang3.StringUtils;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.ChildData;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;
import org.apache.curator.framework.state.ConnectionState;
import org.apache.curator.framework.state.ConnectionStateListener;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.utils.CloseableUtils;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.KeeperException;

import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

/***
 * 使用zk实现分布式独占锁
 */
public class ZkOnlyLock {
    private static String lockNameSpace = "/mylock";
    public static String CONNECT_ADDR = "localhost:2181";
    private static final ThreadLocal<String> threadUuid = new ThreadLocal<String>() {
        @Override
        protected String initialValue() {
            return UUID.randomUUID().toString();
        }
    };
    private CuratorFramework cf;
    private String locakPath;

    /***
     *
     * @param lockPath 锁路径
     */
    public ZkOnlyLock(String lockPath) {
        this.locakPath = lockNameSpace + "/" + lockPath;
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        cf = CuratorFrameworkFactory.builder()
                .connectString(CONNECT_ADDR)
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(5000)
                .retryPolicy(retryPolicy)
                .build();
        cf.getConnectionStateListenable().addListener(new ConnectionStateListener() {
            @Override
            public void stateChanged(CuratorFramework curatorFramework, ConnectionState connectionState) {
                if (connectionState == ConnectionState.LOST) {
                    System.out.println("连接丢失");//连接丢失
                } else if (connectionState == ConnectionState.CONNECTED) {
                    System.out.println("成功连接");
                } else if (connectionState == ConnectionState.RECONNECTED) {
                    System.out.println("重连成功");
                }
            }
        });
        cf.start();
    }

    public void lock() {
        this.lock(0, null);
    }

    public void lock(long millisToWait, TimeUnit unit) {
        boolean doDeleteOurPath = false;
        for (;doDeleteOurPath == false;) {
            try {
                String path = cf.create()
                        .creatingParentsIfNeeded() //自动创建父节点
                        .withMode(CreateMode.PERSISTENT)
                        .forPath(locakPath, threadUuid.get().getBytes());
                if (StringUtils.isNoneBlank(path)) {
                    System.out.println("线程" + Thread.currentThread().getId() + "获得锁");
                    break;
                }
            } catch (Exception e) {
                if (e instanceof KeeperException.NodeExistsException) {
                    //如果是已经存在节点了，则设置watch并阻塞
                    NodeCache cache = new NodeCache(cf, locakPath);
                    CountDownLatch countDownLatch = new CountDownLatch(1);
                    cache.getListenable().addListener(new NodeCacheListener() {
                        @Override
                        public void nodeChanged() throws Exception {
                            ChildData childData = cache.getCurrentData();
                            if (childData == null) {
                                System.out.println("===节点被删除，可以开始尝试创建===");
                                countDownLatch.countDown();
                            }
                        }
                    });
                    try {
                        cache.start();
                    } catch (Exception exception) {
                        exception.printStackTrace();
                    }
                    try {
                        if (unit != null) {
                            countDownLatch.await(millisToWait, unit);
                            doDeleteOurPath = true;
                        } else {
                            countDownLatch.await();
                        }
                        CloseableUtils.closeQuietly(cache);
                    } catch (InterruptedException interruptedException) {
                        interruptedException.printStackTrace();
                    }
                }
            }
        }
    }

    /***
     * 解锁
     */
    public void unlock() {
        try {
            String uuidData = new String(cf.getData().forPath(locakPath), "UTF-8");
            //如果存的是同一个uuid则进行删除，否则不是自己加锁的
            if (StringUtils.isNoneBlank(uuidData) && uuidData.equals(threadUuid.get())) {
                cf.delete().forPath(locakPath);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            threadUuid.remove();
        }
    }
}
```

**测试代码**

```java
/****
     * 测试加锁
     */
    @Test
    public void onlyLockTest1() {
        ZkOnlyLock zkOnlyLock = new ZkOnlyLock("lock001");
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        for (int i = 0; i < 5; i++) {
            executorService.execute(() -> {
                try {
                    zkOnlyLock.lock();
                    System.out.println("做事ing");
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } finally {
                    zkOnlyLock.unlock();
                }
            });
        }
    }

    /***
     * 测试超时
     * @throws InterruptedException
     */
    @Test
    public void onlyLockTest2() throws InterruptedException {
        ZkOnlyLock zkOnlyLock = new ZkOnlyLock("lock001");
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                zkOnlyLock.lock(10,TimeUnit.SECONDS);
                System.out.println("====do=====");
                zkOnlyLock.unlock();
            }
        });
        thread.start();
        zkOnlyLock.lock();
        TimeUnit.SECONDS.sleep(30);
        zkOnlyLock.unlock();
        thread.join();
    }
```

### 缺点

这种方式的锁达不到公平的效果，每次一个锁释放后其他的锁就会同时触发watch。造成同一时间点多次请求Zookeeper。有性能问题。

## 独占式公平分布式锁

![](zookeeper-lock/2.png)

![](zookeeper-lock/3.png)

1. 所有客户端创建自己的锁节点
2. 从 Zookeeper 端获取 /share_lock下所有的子节点
3. 判断自己创建的锁节点是否可以获取锁(如果是第一个就是可以获取到锁)，如果可以，持有锁。否则对自己上一个节点设置watcher
4. 持有锁的客户端删除自己的锁节点，某个客户端收到该节点被删除的通知，并获取锁
5. 重复步骤4，直至无客户端在等待获取锁了

### 代码实例

待续

## 参考

- https://www.jianshu.com/p/6e20e65f301a
- http://www.tianxiaobo.com/2018/01/20/%E5%9F%BA%E4%BA%8E-Zookeeper-%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E5%AE%9E%E7%8E%B0/
- https://www.jianshu.com/p/7fe495c93290