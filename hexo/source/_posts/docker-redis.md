---
title: Docker实战测试
tags:
  - Docker
categories:  Docker
description : Docker实战测试
date: 2019-07-18 10:00:00
---
<!--more-->

### Docker搭建Redis集群

***1.创建Docker自定义网络***

```shell
docker network create redis-cluster-net #创建自定义网络保证集群内的Redis能通过容器名ping得通
[root@localhost ~]# docker network inspect redis-cluster-net #查看创建的自定义网络的GateWay
"IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"
                }
            ]
}
```

***2.通过创建6个Redis对应的数据卷（data和redis.conf）***

```shell
for port in $(seq 1 6);\
do \
mkdir -p /home/project/redis/node-${port}/conf 
touch /home/project/redis/node-${port}/conf/redis.conf 
cat << EOF >> /home/project/redis/node-${port}/conf/redis.conf 
port 6379 
bind 0.0.0.0
cluster-enabled yes 
cluster-config-file nodes.conf 
cluster-node-timeout 5000 
cluster-announce-ip 172.20.0.1${port} 
cluster-announce-port 6379 
cluster-announce-bus-port 16379 
appendonly yes
EOF
done
```

***3.启动6个redis***

```shell
下面几步可以通过脚本来做，为了容易理解，我先不用脚本

docker run -p 6371:6379 -p 16671:16379 --name redis-1 -v /home/project/redis/node-1/data:/data -v /home/project/redis/node-1/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.11 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf


docker run -p 6372:6379 -p 16672:16379 --name redis-2 -v /home/project/redis/node-2/data:/data -v /home/project/redis/node-2/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.12 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf


docker run -p 6373:6379 -p 16673:16379 --name redis-3 -v /home/project/redis/node-3/data:/data -v /home/project/redis/node-3/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.13 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf


docker run -p 6374:6379 -p 16674:16379 --name redis-4 -v /home/project/redis/node-4/data:/data -v /home/project/redis/node-4/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.14 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf


docker run -p 6375:6379 -p 16675:16379 --name redis-5 -v /home/project/redis/node-5/data:/data -v /home/project/redis/node-5/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.15 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf


docker run -p 6376:6379 -p 16676:16379 --name redis-6 -v /home/project/redis/node-6/data:/data -v /home/project/redis/node-6/conf/redis.conf:/etc/redis/redis.conf -d --net redis-cluster-net --ip 172.20.0.16 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
```

结果如下，6台redis都已启动，且能相互ping通

```shell
[root@localhost ~]# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                                                      NAMES
9547f6533dd7   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   13 minutes ago   Up 13 minutes   0.0.0.0:6376->6379/tcp, :::6376->6379/tcp, 0.0.0.0:16676->16379/tcp, :::16676->16379/tcp   redis-6
379c980981ae   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes   0.0.0.0:6375->6379/tcp, :::6375->6379/tcp, 0.0.0.0:16675->16379/tcp, :::16675->16379/tcp   redis-5
d61aaa73d81d   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes   0.0.0.0:6374->6379/tcp, :::6374->6379/tcp, 0.0.0.0:16674->16379/tcp, :::16674->16379/tcp   redis-4
314127dbe25d   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes   0.0.0.0:6373->6379/tcp, :::6373->6379/tcp, 0.0.0.0:16673->16379/tcp, :::16673->16379/tcp   redis-3
ad87d3457967   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes   0.0.0.0:6372->6379/tcp, :::6372->6379/tcp, 0.0.0.0:16672->16379/tcp, :::16672->16379/tcp   redis-2
67031339409f   redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   15 minutes ago   Up 15 minutes   0.0.0.0:6371->6379/tcp, :::6371->6379/tcp, 0.0.0.0:16671->16379/tcp, :::16671->16379/tcp   redis-1
```
***4.设置集群***
```shell
#首先进入其中一个redis
docker exec -it redis-1 /bin/sh #redis默认没有bash
#配置集群
redis-cli --cluster create 172.20.0.11:6379 172.20.0.12:6379 172.20.0.13:6379 172.20.0.14:6379 172.20.0.15:6379 172.20.0.16:6379 --cluster-replicas 1
#接下去就是按照redis集群搭建的提示进行处理了
```


