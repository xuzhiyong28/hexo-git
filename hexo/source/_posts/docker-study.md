---
title: Docker学习笔记
tags:
	- Docker
categories:  Docker
description : Docker学习笔记
date: 2019-07-18 10:00:00

---

<!--more-->

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
docker run [可选参数] image:tag | docker container run [可选参数] image:tagdo
#参书说明 
--name="Name" 容器名字 tomcat01 tomcat02 用来区分容器 
-d 后台方式运行 
-it 使用交互方式运行，进入容器查看内容 
-p 指定容器的端口 -p 8080(宿主机):8080(容器) 
-P(大写) 随机指定端口
docker run -it images:tag /bin/bash

#进入容器
docker exec -it 容器id/名称 /bin/bash
#停止，启动，重启，kill容器
docker stop/start/restart/kill 容器id/名称
#删除容器，此容器必须没在运行
docker rm 容器id/名称
#查看容器
docker inspect 容器id/名称
# 删除所有容器
docker rm -f $(docker ps -aq)
# 获取容器日志
docker logs [-f 跟踪日志输出 -t 显示时间戳 --tail 仅列出最新N条容器日志] 容器ID/名称
# 删除volume
ls -l /var/lib/docker/volumes|grep -Ev "metadata|backingFsBlockDev|grep"|awk '{print $NF}'|xargs rm -fr
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

# 校验Compose文件格式是否正确，若正确则显示配置，若格式错误显示错误原因
docker-compose -f docker-compose.yml config

# 列出docker-compose.yml所包含的镜像
docker-compose [-f docker-compose.yml] images

# 列出项目中目前的所有容器
docker-compose [-f docker-compose.yml] skywalking.yml ps

# 查看服务容器的输出
docker-compose [-f docker-compose.yml] logs [elasticsearch]

#常用选项
-f file : 指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定
-p name : NAME 指定项目名称，默认将使用所在目录名称作为项目名
–verbose :  输出更多调试信息
```

### Docker-Compose模板

```yaml
version: "3.3"
networks:
  ems:
    driver: bridge

services:
  mysqldb:
    image: mysql:5.7.19
    container_name: mysql
    ports:
      - "3306:3306"
    volumes:
      - /home/docker-compose-demo/demo1/mysql/conf:/etc/mysql/conf.d
      - /home/docker-compose-demo/demo1/mysql/logs:/logs
      - /home/docker-compose-demo/demo1/mysql/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - ems
    depends_on:
      - redis
      
  redis:
    image: redis:4.0.14
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - ems
    volumes:
      - /home/docker-compose-demo/demo1/redis/data:/data
    command: redis-server
```

```yaml
version: '3.5'
services:
  nacos1:
    restart: always
    # 指定使用的镜像,或者不使用image,而是使用build来指定Dockerfile的路径
    image: nacos/nacos-server:${NACOS_VERSION}
    container_name: nacos1
    #用来给容器root权限
    privileged: true
    # 暴露端口信息， 宿主机端口:容器端口
    # ports和expose有区别，与posts不同的是expose只可以暴露端口而不能映射到主机，只供外部服务连接使用；仅可以指定内部端口为参数
    ports:
     - "8001:8001"
     - "8011:9555"
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 1024M 
    # 从文件中获取环境变量
    env_file: 
     - ./nacos.env
    # 设置环境变量 
    environment:
        NACOS_SERVER_IP: ${NACOS_SERVER_IP_1}
        NACOS_APPLICATION_PORT: 8001
        NACOS_SERVERS: ${NACOS_SERVERS}
    # 设置卷挂载的路径 宿主机路径:容器路径    
    volumes:
     - ./logs_01/:/home/nacos/logs/
     - ./data_01/:/home/nacos/data/
     - ./config/:/home/nacos/config/
    networks:
      - ha-network-overlay
  nacos2:
    restart: always
    image: nacos/nacos-server:${NACOS_VERSION}
    container_name: nacos2
    privileged: true
    ports:
     - "8002:8002"
     - "8012:9555"
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 1024M    
    env_file: 
     - ./nacos.env     
    environment:
        NACOS_SERVER_IP: ${NACOS_SERVER_IP_2}
        NACOS_APPLICATION_PORT: 8002
        NACOS_SERVERS: ${NACOS_SERVERS}
    volumes:
     - ./logs_02/:/home/nacos/logs/
     - ./data_02/:/home/nacos/data/
     - ./config/:/home/nacos/config/
    networks:
      - ha-network-overlay
  nacos3:
    restart: always
    image: nacos/nacos-server:${NACOS_VERSION}
    container_name: nacos3
    privileged: true
    ports:
     - "8003:8003"
     - "8013:9555"
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 1024M    
    env_file: 
     - ./nacos.env 
    environment:
        NACOS_SERVER_IP: ${NACOS_SERVER_IP_3}
        NACOS_APPLICATION_PORT: 8003
        NACOS_SERVERS: ${NACOS_SERVERS}         
    volumes:
     - ./logs_03/:/home/nacos/logs/
     - ./data_03/:/home/nacos/data/
     - ./config/:/home/nacos/config/
    networks:
      - ha-network-overlay
networks:
   ha-network-overlay:
     external: true
```



### 参考

- https://developer.aliyun.com/article/825708#slide-10