
---
title: "Docker实战（二十二）Docker-Compose部署Zookeeper集群环境"
date: 2016-11-04 10:44:23
tags: [Docker环境]
categories: [Docker]
---

本篇我们具体使用Docker-Compose来部署Zookeeper集群环境，这里我们使用Zookeeper官方提供的Docker镜像来搭建集群环境，官方的镜像地址：https://hub.docker.com/_/zookeeper/

#### 下载Zookeeper官方的Docker镜像

```
$ docker pull zookeeper:latest
```

#### zoo.cfg配置文件

这里我们将部署三台Docker容器组成一个Zookeeper集群，然后我们在本地创建一个zoo.cfg配置文件，指定好Zookeeper集群的配置，zk1，zk2，zk3分别是三台Zookeeper服务器的host名称

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/data
clientPort=2181
dataLogDir=/opt/log
```

#### docker-compose.yml配置文件

```
version: '2'
services:
   zoo1:
      # 指定当前构建的Docker容器的镜像
      image: zookeeper
      restart: always
      # 指定当前构建的Docker容器的名称
      container_name: zk1
      networks:
         zoo_net:
            # 指定当前构建的Docker容器的IP地址
            ipv4_address: 172.18.0.2
      # 指定当前构建的Docker容器的host配置
      extra_hosts:
         - "zoo1:172.18.0.2"
         - "zoo2:172.18.0.3"
         - "zoo3:172.18.0.4"
      # 指定当前构建的Docker容器的volume挂在目录设置
      volumes:
         - ~/Downloads/yunyu/zookeeper_docker/data/zoo1:/opt/data
         - ~/Downloads/yunyu/zookeeper_docker/logs/zoo1:/opt/log
      # 指定当前构建的Docker容器对外开放的端口号映射
      ports:
         - "2181:2181"
         - "2881:2888"
         - "3881:3888"
      # 指定当前构建的Docker容器环境变量设置
      environment:
         ZOO_MY_ID: 1
         ZOO_SERVERS: server.1=zoo1:2881:3881 server.2=zoo2:2882:3882 server.3=zoo3:2883:3883

   zoo2:
      image: zookeeper
      restart: always
      container_name: zk2
      networks:
         zoo_net:
            ipv4_address: 172.18.0.3
      extra_hosts:
         - "zoo1:172.18.0.2"
         - "zoo2:172.18.0.3"
         - "zoo3:172.18.0.4"
      volumes:
         - ~/Downloads/yunyu/zookeeper_docker/data/zoo2:/opt/data
         - ~/Downloads/yunyu/zookeeper_docker/logs/zoo2:/opt/log
      ports:
         - "2182:2181"
         - "2882:2888"
         - "3882:3888"
      environment:
         ZOO_MY_ID: 2
         ZOO_SERVERS: server.1=zoo1:2881:3881 server.2=zoo2:2882:3882 server.3=zoo3:2883:3883

   zoo3:
      image: zookeeper
      restart: always
      container_name: zk3
      networks:
         zoo_net:
            ipv4_address: 172.18.0.4
      extra_hosts:
         - "zoo1:172.18.0.2"
         - "zoo2:172.18.0.3"
         - "zoo3:172.18.0.4"
      volumes:
         - ~/Downloads/yunyu/zookeeper_docker/data/zoo3:/opt/data
         - ~/Downloads/yunyu/zookeeper_docker/logs/zoo3:/opt/log
      ports:
         - "2183:2181"
         - "2883:2888"
         - "3883:3888"
      environment:
         ZOO_MY_ID: 3
         ZOO_SERVERS: server.1=zoo1:2881:3881 server.2=zoo2:2882:3882 server.3=zoo3:2883:3883

networks:
  zoo_net:
    driver: bridge
    ipam:
      driver: default
      config:
      - subnet: 172.18.0.0/16
        gateway: 172.18.0.1
```

这里需要注意几点

1. 因为我们是单机部署了多个Docker容器模拟Zookeeper集群的，所以需要做端口映射2181，2888，3888。需要将2888和3888端口也暴露出来，因为如果不暴露出来一旦leader几点挂了，其他follower无法再次进行选举，因为选举是通过3888端口进行的
2. 这里我们指定好了Docker容器的IP地址，这样不会动态的去获取IP地址导致每次启动Docker容器IP地址都会变化
3. 需要设置/etc/hosts配置文件中的host配置
4. 将Docker容器中的/opt/data和/opt/log目录挂在到宿主机的指定目录下
5. 设置了一个网卡zoo_net，网段是172.18.0.0，网关是172.18.0.1
6. Docker-Compose的version 2版本语法有些变化，注意检查一下networks的配置，否则启动docker-compose up会无法启动

#### 启动Docker-Compose

```
# 启动Docker-Compose后，会自动创建Docker容器并且启动
$ docker-compose up

# 查看当前正在运行的Docker容器
$ docker-compose ps
Name              Command               State                                   Ports
----------------------------------------------------------------------------------------------------------------------
zk1    /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2181->2181/tcp, 0.0.0.0:2881->2888/tcp, 0.0.0.0:3881->3888/tcp
zk2    /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2182->2181/tcp, 0.0.0.0:2882->2888/tcp, 0.0.0.0:3882->3888/tcp
zk3    /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2183->2181/tcp, 0.0.0.0:2883->2888/tcp, 0.0.0.0:3883->3888/tcp
```

#### 验证Zookeeper集群的可用性

```
# 分别进入到正在运行的三个Docker容器中
$ docker exec -it 486110828ff1 /bin/bash

# 进入Zookeeper的安装目录，这里要参考Zookeeper官方的Dockerfile文件配置
$ cd /zookeeper-3.4.9/bin

# 检查Zookeeper的状态（三个Docker容器的状态都不一样，只有一个leader，另外两个是follower）
$ zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Mode: follower

# 分别监听三个Docker容器的2181端口情况，都是正在监听状态
$ netstat -anp | grep 2181
tcp        0      0 :::2181                 :::*                    LISTEN      -

# 然后通过宿主机检查2181, 2182, 2183三个端口（这里连接的是2182端口）
$ telnet 10.10.1.66 2182
Trying 10.10.1.66...
Connected to localhost.
Escape character is '^]'.

# telnet能够连接说明2182端口，同时检查对应2182端口的Docker容器会创建一个TCP连接如下，说明连接都正常了
$ netstat -anp | grep 2181
tcp        0      0 :::2181                 :::*                    LISTEN      -
tcp        0      0 ::ffff:172.18.0.3:2181  ::ffff:172.18.0.1:48996 ESTABLISHED -

# 检查重新Zookeeper的重新选举功能
# 停止leader的Docker容器
$ docker stop 486110828ff1

# 再分别查看另外两个Docker容器的Zookeeper服务状态，其中一个会被选举成leader，另外一个还是follower
$ zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Mode: leader
```

到此为止，我们的使用Zookeeper官方Docker镜像搭建Zookeeper集群已经完成了

参考文章：

- https://docs.docker.com/compose/compose-file/#/network-configuration-reference
- https://docs.docker.com/compose/networking/
- https://hub.docker.com/_/zookeeper/
- https://github.com/31z4/zookeeper-docker/blob/7e7eac6d6c11428849ec13bb7d240e4cfa21b2e7/3.4.9/Dockerfile
- https://github.com/31z4/zookeeper-docker/tree/7e7eac6d6c11428849ec13bb7d240e4cfa21b2e7
- http://blog.csdn.net/cuisongliu/article/details/51817203