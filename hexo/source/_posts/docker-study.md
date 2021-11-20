---
title: Docker学习笔记
tags:
  - Docker
categories:  Docker
description : Docker学习笔记
date: 2019-07-18 10:00:00
---
### Docker安装

```sh
#卸载
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

#需要的安装包
yum install -y yum-utils

#国内镜像源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#更新yum软件包索引 
yum makecache fast

#安装docker相关的 docker-ce 社区版 而ee是企业版 
yum install docker-ce docker-ce-cli containerd.io

#使用docker version查看是否按照成功 
docker version

```

### Docker镜像常用命令

```shell
docker image pull xxx #从仓库拉取镜像
docker images #查看所有镜像
docker image inspect xxx #查看镜像具体信息
docker rmi xxx #删除镜像  docker rmi $(docker images -aq) 删除所有镜像
docker build -f Dockerfile -t 镜像名:[tag] . #构建一个镜像
```

### Docker容器常用命令

```shell
#从image创建容器并启动
docker run [可选参数] image | docker container run [可选参数] image
#参书说明 
--name="Name" 容器名字 tomcat01 tomcat02 用来区分容器 
-d 后台方式运行 
-it 使用交互方式运行，进入容器查看内容 
-p 指定容器的端口 -p 8080(宿主机):8080(容器) 
-P(大写) 随机指定端口

#进入容器
docker -it 容器id/名称 /bin/bash
#停止，启动，重启，kill容器
docker stop/start/restart/kill 容器id/名称
#删除容器，此容器必须没在运行
docker rm 容器id/名称
#查看容器
docker inspect 容器id/名称
```



### Docker-Compose常用命令

```shell
#用于部署一个 Compose 应用,默认情况下该命令会读取名为 docker-compose.yml 或 docker-compose.yaml 的文件
#如果名字不是上面两个，则使用-f指定
docker-compose up

#停止 Compose 应用相关的所有容器，但不会删除它们。
docker-compose stop

#用于删除已停止的 Compose 应用
docker-compose rm

#重启已停止的 Compose 应用
docker-compose restart

#用于列出 Compose 应用中的各个容器
docker-compose ps

#输出内容包括当前状态、容器运行的命令以及网络端口，但是不会删除卷和镜像
docker-compose down
```

