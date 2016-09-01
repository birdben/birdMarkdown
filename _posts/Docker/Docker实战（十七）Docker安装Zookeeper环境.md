---
title: "Docker实战（十七）Docker安装Zookeeper环境"
date: 2016-09-01 14:50:49
tags: [Docker命令, Dockerfile, Zookeeper]
categories: [Docker]
---

##### Zookeeper安装

```
# 下载Zookeeper
$ wget http://apache.fayea.com/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz

# 解压Zookeeper压缩包
$ tar -zxvf zookeeper-3.4.8.tar.gz

# 需要修改下面Zookeeper相关的配置文件
```

##### Dockerfile文件
```
############################################
# version : birdben/zookeeper:v1
# desc : 当前版本安装的zookeeper
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

# 容器需要开放Zookeeper 2181端口
EXPOSE 2181

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/zookeeper/Dockerfile

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
# The number of milliseconds of each ticktickTime=2000# The number of ticks that the initial # synchronization phase can takeinitLimit=10# The number of ticks that can pass between # sending a request and getting an acknowledgementsyncLimit=5# the directory where the snapshot is stored.# do not use /tmp for storage, /tmp here is just # example sakes.dataDir=/var/zookeeper# the port at which the clients will connectclientPort=2181# the maximum number of client connections.# increase this if you need to handle more clients#maxClientCnxns=60## Be sure to read the maintenance section of the # administrator guide before turning on autopurge.## http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance## The number of snapshots to retain in dataDir#autopurge.snapRetainCount=3# Purge task interval in hours# Set to "0" to disable auto purge feature#autopurge.purgeInterval=1
```


##### 控制台终端

```
# 构建镜像
$ docker build -t "birdben/zookeeper:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 2181:2181 -t -i "birdben/zookeeper:v1"
```

##### supervisor无法监控zookeeper

```
supervisor启动的程序必须是非daemon的启动方式，这里找到zookeeper的启动脚本${ZOOKEEPER_HOME}/bin/zkServer.sh，正常的参数可以选择start, status, stop等等。这里我们找到参数start-foreground，这个就是非守护进程的方式启动。所以supervisord.conf配置文件修改成下面的方式
${ZOOKEEPER_HOME}/bin/zkServer.sh start-foreground
```

##### 使用zkCli.sh连接服务端进行操作

```
# 我们通过另一个zk客户端连接到Docker容器的zk服务端
$ ./zkCli.sh -server 10.10.1.167:2181
Connecting to 10.10.1.167:2181
2016-09-01 14:32:42,398 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.8--1, built on 02/06/2016 03:18 GMT
2016-09-01 14:32:42,402 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=localhost
2016-09-01 14:32:42,402 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.7.0_79
2016-09-01 14:32:42,405 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2016-09-01 14:32:42,405 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/Users/yunyu/dev/zookeeper-3.4.8/bin/../build/classes:/Users/yunyu/dev/zookeeper-3.4.8/bin/../build/lib/*.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/Users/yunyu/dev/zookeeper-3.4.8/bin/../conf:
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/Users/yunyu/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/var/folders/0h/jtjrr7g95mv2pt4ts1tgmzyh0000gn/T/
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Mac OS X
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=x86_64
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=10.11.5
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=yunyu
2016-09-01 14:32:42,406 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/Users/yunyu
2016-09-01 14:32:42,407 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/Users/yunyu/dev/zookeeper-3.4.8/bin
2016-09-01 14:32:42,408 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=10.10.1.167:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@3823ed31
Welcome to ZooKeeper!
2016-09-01 14:32:42,441 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server 10.10.1.167/10.10.1.167:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2016-09-01 14:32:42,453 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@876] - Socket connection established to 10.10.1.167/10.10.1.167:2181, initiating session
2016-09-01 14:32:42,492 [myid:] - INFO  [main-SendThread(10.10.1.167:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server 10.10.1.167/10.10.1.167:2181, sessionid = 0x156e4682ae30000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 10.10.1.167:2181(CONNECTED) 0]

# 查看znode节点[zk: 10.10.1.167:2181(CONNECTED) 0] ls
[zookeeper]

# 创建新的znode节点，关联到"my_data"
[zk: 10.10.1.167:2181(CONNECTED) 3] create /zk_test my_dataCreated /zk_test[zk: 10.10.1.167:2181(CONNECTED) 4] ls /
[zookeeper, zk_test]
```

##### 遇到问题zkServer.sh status

```
# SSH登录到Docker容器，查看supervisor的状态
$ sudo supervisorctl status
sshd                             RUNNING    pid 8, uptime 0:35:36
zookeeper                        RUNNING    pid 9, uptime 0:35:36

# 查看zookeeper进程
$ ps -ef | grep zookeeper
root         9     1  0 06:20 ?        00:00:04 /software/jdk7/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /software/zookeeper-3.4.8/bin/../build/classes:/software/zookeeper-3.4.8/bin/../build/lib/*.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/software/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/software/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/software/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/software/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/software/zookeeper-3.4.8/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /software/zookeeper-3.4.8/bin/../conf/zoo.cfg
admin      198   191  0 06:57 pts/0    00:00:00 grep zookeeper

# 使用zkServer.sh status查看zk的状态，发现提示zk并没有运行
$ ./zkServer.sh status
bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
ZooKeeper JMX enabled by default
Using config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.

# 这个问题是因为zk需要依赖jdk环境，我们需要配置java环境变量，因为Dockerfile中的配置对于我们ssh的admin用户是不生效的，需要单独export一下
$ export JAVA_HOME=/software/jdk7

# 再次查看就OK了
$ ./zkServer.sh status
bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
ZooKeeper JMX enabled by default
Using config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: standalone
```


参考文章：

- http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html