---
title: "Docker实战（十二）Docker安装ElasticSearch集群环境"
date: 2016-01-15 02:14:25
tags: [Docker命令, Dockerfile, ElasticSearch]
categories: [Docker]
---

ES集群需要配置几个地方，最简单的是配置集群名称和Node名称，其他的基本使用默认值就可以了，下面是elasticsearch.yml配置文件部分属性的用法，可以参考使用

```
# 集群名称，默认为elasticsearch
cluster.name: birdESCluster

# 节点Node名称，ES启动时会自动创建节点名称，默认的是从ES的，但你也可进行配置
node.name: "birdESNode01"

# 是否作为主节点，每个节点都可以被配置成为主节点，默认值为true
node.master: true

# 是否存储数据，即存储索引片段，默认值为true
node.data: true

# master和data同时配置会产生一些奇异的效果：
1) 当master为false，而data为true时，会对该节点产生严重负荷；
2) 当master为true，而data为false时，该节点作为一个协调者；
3) 当master为false，data也为false时，该节点就变成了一个负载均衡器。
你可以通过连接来查看集群状态
http://localhost:9200/_cluster/health
http://localhost:9200/_cluster/nodes
或者使用插件来查看集群状态
http://github.com/lukas-vlcek/bigdesk
http://mobz.github.com/elasticsearch-head

# 设置一个索引的碎片数量，默认值为5
index.number_of_shards: 5

# 设置一个索引可被复制的数量，默认值为1
index.number_of_replicas: 1

# 当你想要禁用公布式时，你可以进行如下设置：
index.number_of_shards: 1
index.number_of_replicas: 0

这两个属性的设置直接影响集群中索引和搜索操作的执行。假设你有足够的机器来持有碎片和复制品，那么可以按如下规则设置这两个值：
1) 拥有更多的碎片可以提升索引执行能力，并允许通过机器分发一个大型的索引；
2) 拥有更多的复制器能够提升搜索执行能力以及集群能力。
对于一个索引来说，number_of_shards只能设置一次，而number_of_replicas可以使用索引更新设置API在任何时候被增加或者减少。
ElasticSearch关注加载均衡、迁移、从节点聚集结果等等。可以尝试多种设计来完成这些功能。
可以连接http://localhost:9200/A/_status来检测索引的状态。

# 配置文件所在的位置，即elasticsearch.yml和logging.yml所在的位置
path.conf: /path/to/conf

# 分配给当前节点的索引数据所在的位置
path.data: /path/to/data

# 可以可选择的包含一个以上的位置，使得数据在文件级别跨越位置，这样在创建时就有更多的自由路径
path.data: /path/to/data1,/path/to/data2

# 日志文件所在位置
path.logs: /path/to/logs

# 临时文件位置
path.work: /path/to/work

# 插件安装位置
path.plugins: /path/to/plugins

#  默认情况下，ElasticSearch使用0.0.0.0地址（也就是说如果这个机子有几个网卡，则ElasticSearch都可以通过这些IP来使用其服务。所以如果我们的服务器有网卡绑定在外网时一定要注意设置ElasticSearch的属性），并为http传输开启9200-9300端口，为节点到节点的通信开启9300-9400端口，也可以自行设置IP地址：
network.bind_host: 192.168.0.1

# publish_host设置其他节点连接此节点的地址，如果不设置的话，则自动获取，publish_host的地址必须为真实地址：
network.publish_host: 192.168.0.1

# bind_host和publish_host可以一起设置：（如果不想外网访问ES，可以设置这个）
network.host: 192.168.0.1

# 可以定制该节点与其他节点交互的端口：
transport.tcp.port: 9300

# 节点间交互时，可以设置是否压缩，转为为不压缩：
transport.tcp.compress: true

# 可以为Http传输监听定制端口：
http.port: 9200

# 设置内容的最大长度：
http.max_content_length: 100mb

# 禁止HTTP
http.enabled: false

# 允许在N个节点启动后恢复过程：
gateway.recover_after_nodes: 1

# 设置初始化恢复过程的超时时间：
gateway.recover_after_time: 5m

# 设置ping其他节点时的超时时间，网络比较慢时可将该值设大：
discovery.zen.ping.timeout: 3s

# 禁止当前节点发现多个集群节点，默认值为true：
discovery.zen.ping.multicast.enabled: false

# 设置新节点被启动时能够发现的主节点列表：
discovery.zen.ping.unicast.hosts: ["10.0.0.28:9300", "10.0.0.28:9500"]

# 设置是否可以通过正则或者_all删除或者关闭索引
action.destructive_requires_name 默认false 允许 可设置true不允许

# 下面是一些查询时的慢日志参数设置
index.search.slowlog.level: TRACE
index.search.slowlog.threshold.query.warn: 10s
index.search.slowlog.threshold.query.info: 5s
index.search.slowlog.threshold.query.debug: 2s
index.search.slowlog.threshold.query.trace: 500ms
index.search.slowlog.threshold.fetch.warn: 1s
index.search.slowlog.threshold.fetch.info: 800ms
index.search.slowlog.threshold.fetch.debug:500ms
index.search.slowlog.threshold.fetch.trace: 200ms

```

##### elasticsearch.yml源文件链接：

- https://github.com/birdben/birdDocker/blob/master/escluster/elasticsearch_01.yml

##### Dockerfile文件
```
############################################
# version : birdben/escluster:v1
# desc : 当前版本安装的escluster
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN echo "export LC_ALL=C"

# 设置 ES 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV ES_HOME /software/elasticsearch-1.7.2

# 复制 elasticsearch-1.7.2 文件到镜像中（elasticsearch-1.7.2文件夹要和Dockerfile文件在同一路径）
ADD elasticsearch-1.7.2 /software/elasticsearch-1.7.2

# 挂载/software/elasticsearch-1.7.2/config和/escluster目录
VOLUME ["/software/elasticsearch-1.7.2/config"]
VOLUME ["/escluster"]

# 容器需要开放ES的9200和9300端口
EXPOSE 9200
EXPOSE 9300

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/escluster/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

# 修改supervisor配置方式如下，修改为不自动重启ES，并且改成非daemon，DFOREGROUND的方式运行，supervisor就可以监控到了
[program:elasticsearch]
command=/bin/bash -c "exec ${ES_HOME}/bin/elasticsearch -DFOREGROUND"
```

##### 控制台终端

```
# 构建镜像
$ docker build -t="birdben/escluster:v1" .
# 执行已经构件好的镜像，这里启动了两个ES的镜像，只是ES的端口号不同，挂载在宿主机器的存储路径也不同
$ docker run -p 9999:22 -p 9200:9200 -p 9300:9300 -v /docker/escluster01:/escluster -v /docker/escluster01/config:/software/elasticsearch-1.7.2/config -t -i 'birdben/escluster:v1'
$ docker run -p 9998:22 -p 9201:9200 -p 9301:9300 -v /docker/escluster02:/escluster -v /docker/escluster02/config:/software/elasticsearch-1.7.2/config -t -i 'birdben/escluster:v1'
```

##### 访问ElasticSearch集群的插件测试

```
# 访问9201和9201端口都能分别访问
http://10.211.55.4:9200/_plugin/head/
http://10.211.55.4:9201/_plugin/head/
# 在Node1上创建索引会自动同步到Node2上，而且对应的data和logs目录也存储在挂载的目录中
```

##### ES集群的访问方式

##### Java API

Elasticsearch为Java用户提供了两种内置客户端：
需要在项目中引入elasticsearch.jar，这个jar中有两种ES提供的Client

##### 节点客户端 （Node Client）

![NodeClient](http://img.blog.csdn.net/20160124142752454?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Instantiating a node based client is the simplest way to get a Client that can execute operations against elasticsearch.

Embedding a node client into your application is the easiest way to connect to an Elasticsearch cluster

Node Client加入ES集群是一个没有数据的Node，就是这个Node不存储任何数据，但是这个Node知道数据存储在ES集群的哪个Node，可以把请求转发到正确的Node上去

注意：

实际上Node Client是在我们的系统，单元测试中启动了一个Embedded ElasticSearch，这个Embedded ElasticSearch是我们不存储任何数据的Node加入ES集群，我们的系统通过它来和ES集群进行通信，最好关闭Embedded Node的http端口（9200端口）防止http请求直接访问，因为Node Client与ES集群Node的通信都是使用的9300端口

```
import static org.elasticsearch.node.NodeBuilder.*;

// on startup

Node node = nodeBuilder().node();
Client client = node.client();

// on shutdown

node.close();

// 设置ES的集群名称，在Java代码中
Node node = nodeBuilder().clusterName("yourclustername").node();
Client client = node.client();

// 也可以设置cluster.name在/src/main/resources/elasticsearch.yml配置文件
```


##### 传输客户端 （Transport Client）

![TransportClient](http://img.blog.csdn.net/20160124142859440?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

The TransportClient connects remotely to an Elasticsearch cluster using the transport module. It does not join the cluster, but simply gets one or more initial transport addresses and communicates with them in round robin fashion on each action (though most actions will probably be "two hop" operations).

Transport Client不会加入集群，只是负责把请求转发给ES集群的Node

Java Client和ES集群通信通过9300端口，使用native ElasticSearch transport protocol， ES集群的Node也使用9300端口进行通信。（如果ES集群的9300端口没有开通，你的Nodes无法构建成一个集群）

注意：Java Client必须和ES集群的Node版本一致（也就是jar包的版本要和ES集群的版本对应一致），否则无法通信

```
// on startup

Client client = TransportClient.builder().build()
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("host1"), 9300))
        .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("host2"), 9300));

// on shutdown

client.close();

// 设置集群名称
Settings settings = Settings.settingsBuilder()
        .put("cluster.name", "myClusterName").build();
Client client = TransportClient.builder().settings(settings).build();
//Add transport addresses and do something with the client...

// 也可以设置cluster.name在/src/main/resources/elasticsearch.yml配置文件，如Node Client
```

注意：addTransportAddress方法

Adds a transport address that will be used to connect to.The Node this transport address represents will be used if its possible to connect to it. If it is unavailable, it will be automatically connected to once it is up.In order to get the list of all the current connected nodes, please see connectedNodes().

当一个ES Node（对应一个transportAddress）不可用时，client会自动发现当前可用的nodes（the current connected nodes），从以下这段代码可知：

TransportClientNodesService

```
int index = randomNodeGenerator.incrementAndGet();
if (index < 0) {
    index = 0;
    randomNodeGenerator.set(0);
}
RetryListener<Response> retryListener = new RetryListener<Response>(callback, listener, nodes, index);
try {
    callback.doWithNode(nodes.get((index) % nodes.size()), retryListener);
} catch (ElasticsearchException e) {
    if (e.unwrapCause() instanceof ConnectTransportException) {
        retryListener.onFailure(e);
    } else {
        throw e;
    }
}
```



##### RESTFUL API

其他语言都可以使用RESTFUL API通过9200端口和ES进行通信，甚至可以通过命令行使用curl命令

HTTP Request格式

```
curl -X<VERB> '<PROTOCOL>://<HOST>/<PATH>?<QUERY_STRING>' -d '<BODY>'

VERB 			: HTTP方法（GET, POST, PUT, HEAD or DELETE）
PROTOCOL 		: HTTP 或者 HTTPS协议（如果有一个HTTPS的代理在ES前端）
HOST 			: ES集群名称或者IP地址
PORT 			: 端口号运行着ES的HTTP服务，默认是9200
QUERY_STRING 	: 查询参数（例如：?pretty）
BODY 			: A JSON-encoded request body
```



参考文章：

- http://my.oschina.net/xiaohui249/blog/228748
- http://blog.csdn.net/geloin/article/details/8444972
- https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/client.html
- https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/node-client.html
- https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html
- http://www.cnblogs.com/huangfox/p/3543134.html
- http://es.xiaoleilu.com/010_Intro/15_API.html
