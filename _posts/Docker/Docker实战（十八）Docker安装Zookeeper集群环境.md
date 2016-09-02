---
title: "Docker实战（十八）Docker安装Zookeeper集群环境"
date: 2016-09-02 10:50:49
tags: [Docker命令, Dockerfile, Zookeeper]
categories: [Docker]
---

##### Dockerfile文件
```
############################################
# version : birdben/zookeeper_cluster:v1
# desc : 当前版本安装的zookeeper_cluster
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 zookeeper 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV ZOOKEEPER_HOME /software/zookeeper-3.4.8
ENV PATH ${ZOOKEEPER_HOME}/bin:$PATH

# 复制 zookeeper-3.4.8 文件到镜像中（zookeeper-3.4.8 文件夹要和Dockerfile文件在同一路径）
ADD zookeeper-3.4.8 /software/zookeeper-3.4.8

# 创建myid文件存储路径
RUN mkdir -p /var/zookeeper/myid

# 授权ZOOKEEPER_HOME路径给admin用户
RUN sudo chown -R admin /software/zookeeper-3.4.8

# 容器需要开放Zookeeper 2181, 2888, 3888端口
EXPOSE 2181
EXPOSE 2888
EXPOSE 3888

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/zookeeper_cluster/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:zookeeper]
command=/bin/bash -c "exec ${ZOOKEEPER_HOME}/bin/zkServer.sh start-foreground"
```

##### 配置ZOOKEEPER_HOME/conf/zoo.cfg并不在conf目录, 需要复制zoo_sample.cfg并改名为zoo.cfg

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/var/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

# 第一个port是从机器（follower）连接到主机器（leader）的端口号，第二个port是进行leadership选举的端口号。
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```


##### 控制台终端

```
# 构建镜像
$ docker build -t "birdben/zookeeper_cluster:v1" .
# 启动Docker容器，这里分别对每个docker容器指定了不同的hostname
# 需要暴露2181客户端连接端口号，否则Docker容器外无法连接到zookeeper集群
$ sudo docker run -p 2181:2181 -h zoo1 --name zoo1 -t -i 'birdben/zookeeper_cluster:v1'
$ sudo docker run -h zoo2 --name zoo2 -t -i 'birdben/zookeeper_cluster:v1'
$ sudo docker run -h zoo3 --name zoo3 -t -i 'birdben/zookeeper_cluster:v1'

# 查询Docker容器对应的IP地址
$ sudo docker inspect --format='{{.NetworkSettings.IPAddress}}' ${CONTAINER_ID}

# 需要exec进入Docker容器配置myid和hosts文件
$ sudo docker exec -it ${CONTAINER_ID} /bin/bash

# 配置每个Docker容器的myid，对应zoo序号执行
$ echo 1 > /var/zookeeper/myid
$ echo 2 > /var/zookeeper/myid
$ echo 3 > /var/zookeeper/myid

# 配置每个Docker容器的/etc/hosts文件
172.17.0.51     zoo1
172.17.0.52     zoo2
172.17.0.53     zoo3

# 分别启动每个Docker容器中的zookeeper服务
$ ./{ZOOKEEPER_HOME}/bin/zkServer.sh start

# 查看每个Docker容器的zookeeper运行状态
$ ./{ZOOKEEPER_HOME}/bin/zkServer.sh status

# 下面是我查看每个zookeeper的状态，zoo2的Docker容器的zk是leader，zoo1和zoo3是follower
root@zoo1:/software/zookeeper-3.4.8/bin# ./zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: follower

root@zoo2:/software/zookeeper-3.4.8/bin# ./zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: leader

root@zoo3:/software/zookeeper-3.4.8/bin# ./zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: follower
```

##### 使用zkCli.sh连接服务端进行操作

```
$ ./zkCli.sh -server 10.10.1.167:2181
Connecting to 10.10.1.167:2181
2016-09-02 11:01:56,761 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.8--1, built on 02/06/2016 03:18 GMT
2016-09-02 11:01:56,764 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=localhost
2016-09-02 11:01:56,764 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.7.0_79
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/Users/yunyu/dev/zookeeper-3.4.8/bin/../build/classes:/Users/yunyu/dev/zookeeper-3.4.8/bin/../build/lib/*.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../conf:
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/Users/yunyu/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/var/folders/0h/jtjrr7g95mv2pt4ts1tgmzyh0000gn/T/
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Mac OS X
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=x86_64
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=10.11.5
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=yunyu
2016-09-02 11:01:56,766 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/Users/yunyu
2016-09-02 11:01:56,767 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/Users/yunyu/dev/zookeeper-3.4.8/bin
2016-09-02 11:01:56,767 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=10.10.1.167:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@5be7d8b1
Welcome to ZooKeeper!
2016-09-02 11:01:56,791 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server 10.10.1.167/10.10.1.167:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2016-09-02 11:01:56,798 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@876] - Socket connection established to 10.10.1.167/10.10.1.167:2181, initiating session
2016-09-02 11:01:56,821 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server 10.10.1.167/10.10.1.167:2181, sessionid = 0x156e8d804300000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 10.10.1.167:2181(CONNECTED) 0]
```

##### 需要注意的地方

- 因为我们zookeeper的启动方式是用的supervisor启动，但是Docker容器启动的时候，我们还不知道Docker容器的IP地址，无法指定hosts文件配置，所以我们要先进入到Docker容器指定好hosts文件配置，然后重新启动zookeeper服务
- myid的配置也是每个Docker容器都不一样，最好跟hosts配置对应
- 需要Docker容器外连接zookeeper集群需要在启动Docker容器时，指定一个Docker容器对外开放2181客户端连接端口号，否则Docker容器外无法连接到zookeeper集群
- 如果查看zookeeper运行状态提示有问题

```
JMX enabled by default
Using config: /root/zookeeper-3.4.6/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.
```
确认下面两点，应该就能排查出问题，我就遇到过重启Docker容器IP
地址变化，导致/etc/hosts中的IP地址配置不正确

- 确认/etc/hosts中是否有各个节点域名解析
- 是否/var/zookeeper/myid有重复值


参考文章：

- https://segmentfault.com/a/1190000003994382
- http://blog.csdn.net/cruise_h/article/details/19046357