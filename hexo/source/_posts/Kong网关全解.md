## Kong

### Kong介绍

​	Kong 是一款基于 OpenResty（Nginx + Lua 模块）编写的高可用、易扩展的，由 Mashape 公司开源的 API Gateway 项目。Kong 是基于 NGINX 和 Apache Cassandra 或 PostgreSQL 构建的，能提供易于使用的 RESTful API 来操作和配置 API 管理系统，所以它可以水平扩展多个 Kong 服务器，通过前置的负载均衡配置把请求均匀地分发到各个 Server，来应对大批量的网络请求。

![](Kong网关全解/2.png)

Kong主要有这几个组件

- Kong Server ：基于 nginx 的服务器，用来接收 API 请求。
- Apache Cassandra/PostgreSQL ：用来存储操作数据。
- Kong dashboard：官方推荐 UI 管理工具，当然，也可以使用 restfull 方式 管理 admin api。

Kong 采用插件机制进行功能定制，插件集（可以是 0 或 N 个）在 API 请求响应循环的生命周期中被执行。插件使用 Lua 编写，目前已有几个基础功能：HTTP 基本认证、密钥认证、CORS（Cross-Origin Resource Sharing，跨域资源共享）、TCP、UDP、文件日志、API 请求限流、请求转发以及 Nginx 监控。

### Kong网关架构

![](Kong网关全解/3.png)

1. Kong 核心基于 OpenResty 构建，实现了请求/响应的 Lua 处理化；
2. Kong 插件拦截请求/响应；
3. Kong Restful 管理 API 提供了 API/API 消费者/插件的管理；
4. 数据中心用于存储 Kong 集群节点信息、API、消费者、插件等信息，目前提供了 PostgreSQL 和 Cassandra 支持，如果需要高可用建议使用 Cassandra；
5. Kong 集群中的节点通过 gossip 协议自动发现其他节点，当通过一个 Kong 节点的管理 API 进行一些变更时也会通知其他节点。每个 Kong 节点的配置信息是会缓存的，如插件，那么当在某一个 Kong 节点修改了插件配置时，需要通知其他节点配置的变更。

## docker安装Kong

这里整理好了使用kong2.8版本的docker-compose安装版本，一键安装，若需要一步一步安装请看文档(https://docs.konghq.com/gateway/2.8.x/install-and-run/docker/)

```yaml
# docker-compose -f docker-compose-kong.yml up -d 启动
# docker-compose -f docker-compose-kong.yml stop  停止服务
# docker-compose -f docker-compose-kong.yml down 停止并删除服务

version: '3.7'

networks:
  kong-net:
    driver: bridge

services:
  # kong数据库
  kong_db:
    image: postgres:9.6
    restart: always  #每次总是启动
    networks:
      - kong-net
    environment:
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: 123456
      POSTGRES_USER: kong
    ports:
      - "5432:5432/tcp"

  # 数据迁移
  kong-migration:
    image: kong/kong-gateway:2.8.4.0-alpine
    command: "kong migrations bootstrap"
    networks: 
      - kong-net
    restart: on-failure
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_DATABASE=kong
      - KONG_PG_PASSWORD=123456
      - KONG_PG_HOST=kong_db
    links: 
      - kong_db #连接的是kong-database服务的
    depends_on:
      - kong_db #依赖于kong-database服务    

  # kong getaway    
  kong:
    image: kong/kong-gateway:2.8.4.0-alpine
    networks:
      - kong-net
    restart: always
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong_db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: 123456
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_ADMIN_GUI_URL: http://localhost:8002
    depends_on:
      - kong-migration
    links: 
      - kong_db
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
      - "8444:8444"
      - "8002:8002"
      - "8445:8445"
      - "8003:8003"
      - "8004:8004"

  #konga gui
  konga-prepare:
    image: pantsel/konga:latest
    command: "-c prepare -a postgres -u postgresql://kong:123456@kong_db:5432/konga"  #注意是用户名：密码@数据库服务名称：端口
    networks:
      - kong-net
    restart: on-failure
    links:
      - kong_db
    depends_on:
      - kong        #依赖kong服务
      - kong_db     #依赖kong-database服务

  konga:
    image: pantsel/konga:latest
    restart: always
    networks:
     - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_HOST: kong_db
      DB_USER: kong
      DB_DATABASE: konga
      DB_PASSWORD: 123456 #必须加上密码，不然会失败
    depends_on:
      - kong
      - kong_db
    ports:
      - "1337:1337"
```

执行完后通过konga默认网址管理kong `http://ip:1337`。 首次进入konga需要注册并添加kong相关配置，这一步百度下，最终的页面:

![](Kong网关全解/1.png)

Kong安装后主要4个端口介绍

- Proxy 8000：接收客户端的 HTTP 请求，并转发到后端的 Upstream。
- Proxy 8443：接收客户端的 HTTPS 请求，并转发到后端的 Upstream。
- Admin 8001：接收管理员的 HTTP 请求，进行 Kong 的管理。
- Admin 8444：接收管理员的 HTTPS 请求，进行 Kong 的管理。

## 网关流程

Kong网关转发流程和组件说明

![](Kong网关全解/4.png)

假设现在有一段nginx负载均衡代码

```
upstream hello_upstream {
	server 192.168.56.20:7001 weight=100;
	server 192.168.56.20:7002 weight=100;
	server 192.168.56.20:7002 weight=100;
}

server {
	listen	80;
	location /app1/ {
		proxy_pass http://hello_upstream;
	}
}
```

按照Kong的组件定义来说，对应关系如下

- service : 表示一个物理服务(host或ip:port)，例如www.baidu.com，也可以表示一个upstream负载均衡,例如上面的hello_upstream
- upstream : 对上上面的hello_upstream，upstream需要配置traget
- target : 翻译过来是目标，也就是上面的`server 192.168.56.20:7001 weight=100`
- route : 路由，每个route都会在一个service下，route与service是N:1的关系

在理解上`upstream`和`target`是比较好理解的，他是对应的nginx的upstream配置，但是`service`和`route`容易混淆，有文章说“route对应nginx的location，service对应nginx的server“，后半句是不对的。service仅仅是定义了一个服务的地址。假设现在我们定义了service(HOST=www.baidu.com)，又定义了一个route(path=/ranran)属于这个service, 当我通过访问`http://localhost:8000/ranran`时，会被转发到www.baidu.com。

这里附上ChatGPT的解释

### chatgpt解释service与route

假设我们有一个Web服务，它暴露了一个RESTful API。我们想使用Kong作为API网关来管理这个API，并且要在Kong中配置这个API的路由。

首先，我们需要将这个Web服务注册为一个Kong Service。在Kong中，一个Service代表了一个后端API服务，它可以是一个HTTP、HTTPS、TCP、UDP等类型的服务。为了将我们的Web服务注册为Kong Service，我们可以使用如下的命令：

```
kotlinCopy code$ curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=my-service' \
  --data 'url=http://my-service.com'
```

在这个例子中，我们将Web服务的名称设置为`my-service`，将它的地址设置为`http://my-service.com`。

接下来，我们需要将这个Service配置到Kong中的一个Route中。在Kong中，一个Route代表了API的一条路由规则，它可以指定URL的匹配规则、HTTP方法、请求头等信息。为了将我们的API配置到Kong中的一个Route中，我们可以使用如下的命令：

```
kotlinCopy code$ curl -i -X POST \
  --url http://localhost:8001/services/my-service/routes \
  --data 'hosts[]=my-api.com' \
  --data 'paths[]=/api'
```

在这个例子中，我们将我们的API配置到了一个名为`my-api.com`的域名下，并且将它的URL路径设置为`/api`。这意味着，当客户端访问`my-api.com/api`时，Kong将会将请求路由到我们注册的`my-service`服务上。

通过这个例子，可以看出，Service和Route之间的关系是非常紧密的。Service代表了我们的后端API服务，而Route则指定了API的访问路径和请求规则。Kong通过这两个概念来管理API网关的路由和负载均衡。

## chatgtp问题

### 如何理解Kong中service的path和route的path他们之间的关系

在Kong中，Service和Route是两个不同的概念，但是它们之间是有关联的。具体来说，Service表示后端的服务，而Route则表示如何将客户端的请求映射到后端的服务。

在Kong中，Service的`path`属性表示后端服务的路径，而Route的`paths`属性表示如何将客户端的请求路径映射到后端服务的路径上。当一个客户端请求到达Kong时，Kong将首先匹配所有的Route，然后将请求转发到与匹配的Route相关联的Service上。在这个过程中，Kong会将Route的路径与其关联的Service的路径进行组合，从而得到最终的请求路径。

举个例子，假设你有一个名为`my-service`的服务，它的路径为`/api`。在Kong中，你可以创建一个与之关联的Route，它的路径为`/server`：

```
POST /routes HTTP/1.1
Host: kong:8001
Content-Type: application/json

{
    "paths": ["/server"],
    "service": {
        "name": "my-service",
        "path": "/api"
    }
}
```

在这个例子中，当一个客户端请求路径为`/server`的请求到达Kong时，Kong将会将其转发到`my-service`服务的`/api/server`路径上。在这个过程中，Kong会自动将Route的路径`/server`与Service的路径`/api`进行组合。

需要注意的是，如果Route的`strip_path`属性设置为`true`，则Kong将会忽略Route的路径，直接将请求转发到Service的路径上。例如，如果你将Route的`strip_path`属性设置为`true`，则客户端请求路径为`/server`的请求将会直接被转发到`my-service`服务的`/api`路径上。

综上所述，Kong中Service的`path`属性表示后端服务的路径，而Route的`paths`属性表示客户端请求路径与后端服务路径的映射关系。当Kong将客户端请求转发到后端服务时，它会自动将Route的路径与Service的路径进行组合，从而得到最终的请求路径。





Regenerate response

## 参考

- https://xie.infoq.cn/article/10e4dab2de0bdb6f2c3c93da6
- https://cloud.tencent.com/developer/article/1110557