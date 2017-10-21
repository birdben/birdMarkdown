---
title: "Docker实战（三十二）Docker安装Kafka集群环境"
date: 2017-07-03 16:02:39
tags: [Docker命令, Dockerfile, Kafka]
categories: [Docker]
---

这里使用docker-compose对之前的Zookeeper和Kafka容器进行编排，构建Kafka集群环境。

##### docker-compose.yml文件
```
version: '2'
services:
   zoo1:
      # 指定当前构建的Docker容器的镜像
      image: birdben/zookeeper:v2
      restart: always
      # 指定当前构建的Docker容器的名称
      container_name: zookeeper1
      networks:
         kafka_net:
            # 指定当前构建的Docker容器的IP地址
            ipv4_address: 172.22.0.2
      # 指定当前构建的Docker容器的host配置
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
      # 指定当前构建的Docker容器的volume挂在目录设置
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo1/conf:/usr/local/zookeeper/conf
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo1/data:/usr/local/zookeeper/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo1/datalogs:/usr/local/zookeeper/datalogs
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo1/logs:/usr/local/zookeeper/logs
      # 指定当前构建的Docker容器对外开放的端口号映射
      ports:
         - "2181:2181"
         - "2881:2888"
         - "3881:3888"

   zoo2:
      image: birdben/zookeeper:v2
      restart: always
      container_name: zookeeper2
      networks:
         kafka_net:
            ipv4_address: 172.22.0.3
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo2/conf:/usr/local/zookeeper/conf
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo2/data:/usr/local/zookeeper/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo2/datalogs:/usr/local/zookeeper/datalogs
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo2/logs:/usr/local/zookeeper/logs
      ports:
         - "2182:2181"
         - "2882:2888"
         - "3882:3888"

   zoo3:
      image: birdben/zookeeper:v2
      restart: always
      container_name: zookeeper3
      networks:
         kafka_net:
            ipv4_address: 172.22.0.4
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo3/conf:/usr/local/zookeeper/conf
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo3/data:/usr/local/zookeeper/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo3/datalogs:/usr/local/zookeeper/datalogs
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/zoo3/logs:/usr/local/zookeeper/logs
      ports:
         - "2183:2181"
         - "2883:2888"
         - "3883:3888"

   kafka1:
      # 指定当前构建的Docker容器的镜像
      image: birdben/kafka:v2
      restart: always
      # 指定当前构建的Docker容器的名称
      container_name: kafka1
      networks:
         kafka_net:
            # 指定当前构建的Docker容器的IP地址
            ipv4_address: 172.22.0.5
      # 指定当前构建的Docker容器的host配置
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
         - "kafka1:172.22.0.5"
         - "kafka2:172.22.0.6"
         - "kafka3:172.22.0.7"
      # 指定当前构建的Docker容器的volume挂在目录设置
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka1/config:/usr/local/kafka/config
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka1/data:/usr/local/kafka/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka1/logs:/usr/local/kafka/logs
      # 指定当前构建的Docker容器对外开放的端口号映射
      ports:
         - "9092:9092"

   kafka2:
      image: birdben/kafka:v2
      restart: always
      container_name: kafka2
      networks:
         kafka_net:
            ipv4_address: 172.22.0.6
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
         - "kafka1:172.22.0.5"
         - "kafka2:172.22.0.6"
         - "kafka3:172.22.0.7"
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka2/config:/usr/local/kafka/config
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka2/data:/usr/local/kafka/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka2/logs:/usr/local/kafka/logs
      ports:
         - "9093:9092"

   kafka3:
      image: birdben/kafka:v2
      restart: always
      container_name: kafka3
      networks:
         kafka_net:
            ipv4_address: 172.22.0.7
      extra_hosts:
         - "zoo1:172.22.0.2"
         - "zoo2:172.22.0.3"
         - "zoo3:172.22.0.4"
         - "kafka1:172.22.0.5"
         - "kafka2:172.22.0.6"
         - "kafka3:172.22.0.7"
      volumes:
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka3/config:/usr/local/kafka/config
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka3/data:/usr/local/kafka/data
         - /Users/yunyu/workspace_git/birdDocker/kafka_cluster/volumes/kafka3/logs:/usr/local/kafka/logs
      ports:
         - "9094:9092"

networks:
  kafka_net:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.22.0.0/16
        gateway: 172.22.0.1

```

##### docker-compose.yml源文件链接：

https://github.com/birdben/birdDocker/blob/master/kafka_cluster/docker-compose.yml

##### Zookeeper的zoo.cfg源文件链接：

https://github.com/birdben/birdDocker/blob/master/kafka_cluster/volumes/zoo1/conf/zoo.cfg

##### Kafka的server.properties源文件链接：

https://github.com/birdben/birdDocker/blob/master/kafka_cluster/volumes/kafka1/config/server.properties

##### 控制台终端

```
# 使用docker-compose启动Kafka集群
$ docker-compose up -d
```

##### 需要注意的地方

Kafka的配置

- listeners：监听列表 - 监听逗号分隔的URL列表和协议。指定hostname为0.0.0.0绑定到所有接口，将hostname留空则绑定到默认接口。合法的listener列表是：PLAINTEXT://myhost:9092,TRACE://:9091 PLAINTEXT://0.0.0.0:9092, TRACE://localhost:9093

- advertised.listeners：发布到Zookeeper供客户端使用监听（如果不同）(是暴露给外部的listeners)。在IaaS环境中，broker可能需要绑定不同的接口。如果没设置，则使用listeners。

- host.name：broker的主机地址。若是设置了，那么会绑定到这个地址上，若是没有，会绑定到所有的接口上，并将其中之一发送到Zookeeper，一般不设置。

如果远端程序想访问Docker容器的Kafka集群，正确的配置方式有3种（推荐方式1和方式2）：

- 方式1：默认配置，设置kafka的server.properties参数host.name为空。
- 方式2：配置advertised.listeners也可以使用IP地址方式，kafka的server.properties添加参数advertised.host.name=宿主机IP
- 方式3：配置advertised.listeners也可以使用hostname方式，需要在docker-compose.yml配置文件中的extra_hosts添加"macbook:172.16.1.147"配置，并且生产者和消费者在连接Kafka集群必须使用macbook

注意：

- 方式1：默认配置
- 方式2：配置的IP地址172.16.1.147是Docker容器Kafka集群的宿主机的局域网IP地址。端口号是docker映射宿主机的端口号，而不是Docker容器内使用的9092端口
- 方式3：配置的hostname为macbook，macbook是docker-compose.yml配置文件中的extra_hosts，添加"macbook:172.16.1.147"配置。并且生产者和消费者也必须使用macbook的hostname来连接kafka，否则是无法生产和消费消息的（使用172.16.1.147也不可以，必须使用macbook）
- 方式4：错误配置，在宿主机中的/etc/hosts配置hostname为macbook，，添加配置"macbook 172.16.1.147"。但是在Docker容器中是无法解析到macbook这个hostname，所以会出现下面的错误
- 方式5：错误配置，配置的hostname为kafka1，kafka2，kafka3是docker-compose.yml配置文件中的container_name。但是这里的kafka1，kafka2，kafka3配置的IP地址都是Docker容器内使用IP地址，无法提供给生产者和消费者使用，所以会出现下面的错误

```
# 方式1：
listeners=PLAINTEXT://:9092

# 方式2：
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://172.16.1.147:9092

# 方式3：（macbook是docker-compose.yml配置文件中的extra_hosts，添加配置"macbook:172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9092

# 方式4：（macbook是配置在宿主机的/etc/hosts配置文件中，添加配置"macbook 172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9092

# 方式5：（kafka1是docker-compose.yml配置文件中的container_name）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka1:9092
```

```
# 方式1：
listeners=PLAINTEXT://:9092

# 方式2：
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://172.16.1.147:9093

# 方式3：（macbook是docker-compose.yml配置文件中的extra_hosts，添加配置"macbook:172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9093

# 方式4：（macbook是配置在宿主机的/etc/hosts配置文件中，添加配置"macbook 172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9093

# 方式5：（kafka2是docker-compose.yml配置文件中的container_name）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka2:9093
```

```
# 方式1：
listeners=PLAINTEXT://:9092

# 方式2：
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://172.16.1.147:9094

# 方式3：（macbook是docker-compose.yml配置文件中的extra_hosts，添加配置"macbook:172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9094

# 方式4：（macbook是配置在宿主机的/etc/hosts配置文件中，添加配置"macbook 172.16.1.147"）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://macbook:9094

# 方式5：（kafka3是docker-compose.yml配置文件中的container_name）
listeners=PLAINTEXT://:9092
advertised.listeners=PLAINTEXT://kafka3:9094
```

注意：如果宿主机是CentOS操作系统，需要注意防火墙是否开启，如果防火墙是开启状态，通过局域网IP地址（172.16.1.147）当前服务器的三个节点之前可能无法互相通信


使用方式4会在controller.log日志中出现如下错误信息，因为在Docker容器中是无法解析到macbook这个hostname。

```
[2017-07-09 05:28:06,583] INFO Disconnecting from macbook:9094 (kafka.producer.SyncProducer)[2017-07-09 05:28:06,583] INFO Fetching metadata from broker BrokerEndPoint(1,macbook,9093) with correlation id 634 for 1 topic(s) Set(host_log) (kafka.client.ClientUtils$)[2017-07-09 05:28:06,583] INFO Connected to macbook:9093 for producing (kafka.producer.SyncProducer)[2017-07-09 05:28:06,583] INFO Disconnecting from macbook:9093 (kafka.producer.SyncProducer)[2017-07-09 05:28:06,583] WARN Fetching topic metadata with correlation id 634 for topics [Set(host_log)] from broker [BrokerEndPoint(1,macbook,9093)] failed (kafka.client.ClientUtils$)java.nio.channels.ClosedChannelException	at kafka.network.BlockingChannel.send(BlockingChannel.scala:110)	at kafka.producer.SyncProducer.liftedTree1$1(SyncProducer.scala:80)	at kafka.producer.SyncProducer.kafka$producer$SyncProducer$$doSend(SyncProducer.scala:79)	at kafka.producer.SyncProducer.send(SyncProducer.scala:124)	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:60)	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:95)	at kafka.consumer.ConsumerFetcherManager$LeaderFinderThread.doWork(ConsumerFetcherManager.scala:68)	at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)[2017-07-09 05:28:06,584] INFO Disconnecting from macbook:9093 (kafka.producer.SyncProducer)[2017-07-09 05:28:06,584] WARN [console-consumer-21678_e920df7081de-1499577947273-1d9469c3-leader-finder-thread], Failed to find leader for Set(host_log-0) (kafka.consumer.ConsumerFetcherManager$LeaderFinderThread)kafka.common.KafkaException: fetching topic metadata for topics [Set(host_log)] from broker [ArrayBuffer(BrokerEndPoint(0,macbook,9092), BrokerEndPoint(2,macbook,9094), BrokerEndPoint(1,macbook,9093))] failed	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:74)	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:95)	at kafka.consumer.ConsumerFetcherManager$LeaderFinderThread.doWork(ConsumerFetcherManager.scala:68)	at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)Caused by: java.nio.channels.ClosedChannelException	at kafka.network.BlockingChannel.send(BlockingChannel.scala:110)	at kafka.producer.SyncProducer.liftedTree1$1(SyncProducer.scala:80)	at kafka.producer.SyncProducer.kafka$producer$SyncProducer$$doSend(SyncProducer.scala:79)	at kafka.producer.SyncProducer.send(SyncProducer.scala:124)	at kafka.client.ClientUtils$.fetchTopicMetadata(ClientUtils.scala:60)	... 3 more
```

使用方式5会在kafka2和kafka3的controller.log日志中出现如下错误信息，因为这里的kafka1，kafka2，kafka3配置的IP地址都是Docker容器内使用IP地址，无法提供给生产者和消费者使用。

```
[2017-07-09 05:34:26,552] WARN [Controller-1-to-broker-1-send-thread], Controller 1's connection to broker kafka2:9093 (id: 1 rack: null) was unsuccessful (kafka.controller.RequestSendThread)java.io.IOException: Connection to kafka2:9093 (id: 1 rack: null) failed	at kafka.utils.NetworkClientBlockingOps$.awaitReady$1(NetworkClientBlockingOps.scala:84)	at kafka.utils.NetworkClientBlockingOps$.blockingReady$extension(NetworkClientBlockingOps.scala:94)	at kafka.controller.RequestSendThread.brokerReady(ControllerChannelManager.scala:232)	at kafka.controller.RequestSendThread.liftedTree1$1(ControllerChannelManager.scala:185)	at kafka.controller.RequestSendThread.doWork(ControllerChannelManager.scala:184)	at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)[2017-07-09 05:34:26,555] WARN [Controller-1-to-broker-2-send-thread], Controller 1's connection to broker kafka3:9094 (id: 2 rack: null) was unsuccessful (kafka.controller.RequestSendThread)java.io.IOException: Connection to kafka3:9094 (id: 2 rack: null) failed	at kafka.utils.NetworkClientBlockingOps$.awaitReady$1(NetworkClientBlockingOps.scala:84)	at kafka.utils.NetworkClientBlockingOps$.blockingReady$extension(NetworkClientBlockingOps.scala:94)	at kafka.controller.RequestSendThread.brokerReady(ControllerChannelManager.scala:232)	at kafka.controller.RequestSendThread.liftedTree1$1(ControllerChannelManager.scala:185)	at kafka.controller.RequestSendThread.doWork(ControllerChannelManager.scala:184)	at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)
```

参考文章：

- https://my.oschina.net/lemonfight/blog/700099
- http://blog.csdn.net/louisliaoxh/article/details/51567515
- https://stackoverflow.com/questions/35861501/kafka-in-docker-not-working
- http://www.cnblogs.com/clonen/p/5284054.html