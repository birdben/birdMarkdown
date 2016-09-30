---
title: "Zookeeper学习（二）Zookeeper集群环境搭建"
date: 2016-09-02 23:54:31
tags: [Zookeeper]
categories: [Hadoop]
---

### ZooKeeper Cluster模式

#### /etc/hosts文件配置
```
172.17.0.51     zoo1
172.17.0.52     zoo2
172.17.0.53     zoo3
```

#### /var/zookeeper/myid文件配置
```
# zoo1的myid配置文件
1

# zoo2的myid配置文件
2

# zoo3的myid配置文件
3
```

#### Zookeeper配置文件
```
# the basic time unit in milliseconds used by ZooKeeper. It is used to do heartbeats and the minimum session timeout will be twice the tickTime.
# tickTime 这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime时间就会发送一个心跳。
tickTime=2000

# the location to store the in-memory database snapshots and, unless specified otherwise, the transaction log of updates to the database.
# dataDir 顾名思义就是Zookeeper保存数据的目录,默认情况下Zookeeper将写数据的日志文件也保存在这个目录里。
dataDir=/var/lib/zookeeper

# the port to listen for client connections
# clientPort 这个端口就是客户端（应用程序）连接Zookeeper服务器的端口，Zookeeper会监听这个端口接受客户端的访问请求。
clientPort=2181

# initLimit 这个配置项是用来配置Zookeeper接受客户端（这里所说的客户端不是用户连接Zookeeper 服务器的客户端，而是Zookeeper服务器集群中连接到Leader的Follower服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过10个心跳的时间（也就是tickTime）长度后Zookeeper服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 10*2000=20 秒。
initLimit=10

# syncLimit 这个配置项标识Leader与Follower之间发送消息，请求和应答时间长度，最长不能超过多少个tickTime的时间长度，总的时间长度就是 5*2000=10 秒。
syncLimit=5

# 第一个port是从机器（follower）连接到主机器（leader）的端口号，第二个port是进行leadership选举的端口号。
# 值得重点注意的一点是，所有三个机器都应该打开端口 2181、2888 和 3888。在本例中，端口 2181 由 ZooKeeper 客户端使用，用于连接到 ZooKeeper 服务器；端口 2888 由对等 ZooKeeper 服务器使用，用于互相通信；而端口 3888 用于领导者选举。您可以选择自己喜欢的任何端口。通常建议在所有 ZooKeeper 服务器上使用相同的端口。
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

#### 启动Zookeeper服务端
```
# 分别启动Hadoop1，Hadoop2，Hadoop3的Zookeeper服务
$ ./bin/zkServer.sh start

ZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgStarting zookeeper ... STARTED

# 检查Hadoop1的Zookeeper服务状态（这里Hadoop1节点的zk是leader，Hadoop2和Hadoop3节点的zk是follower）
$ ./bin/zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /data/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: leader

# 检查Hadoop2的Zookeeper服务状态
$ ./bin/zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /data/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: follower

# 检查Hadoop3的Zookeeper服务状态
$ ./bin/zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /data/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: follower
```

#### 需要注意的地方
```
JMX enabled by default
Using config: /root/zookeeper-3.4.6/bin/../conf/zoo.cfg
Error contacting service. It is probably not running.
```
确认下面两点，应该就能排查出问题，我就遇到过重启Docker容器IP
地址变化，导致/etc/hosts中的IP地址配置不正确

- 确认/etc/hosts中是否有各个节点域名解析
- 是否/var/zookeeper/myid有重复值
- 集群模式只启动一台也会遇到该问题，最好等把其他集群的机器启动好在查看状态

#### 启动Zookeeper Client端

```
# -server：client端连接的IP和端口号
$ ./bin/zkCli.sh -server 127.0.0.1:2181Connecting to 127.0.0.1:21812016-09-30 01:51:22,268 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.8--1, built on 02/06/2016 03:18 GMT2016-09-30 01:51:22,271 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=hadoop12016-09-30 01:51:22,271 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.7.0_792016-09-30 01:51:22,272 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation2016-09-30 01:51:22,272 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/data/jdk1.7.0_79/jre2016-09-30 01:51:22,272 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/data/zookeeper-3.4.8/bin/../build/classes:/data/zookeeper-3.4.8/bin/../build/lib/*.jar:/data/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/data/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/data/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/data/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/data/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/data/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/data/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/data/zookeeper-3.4.8/bin/../conf:2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd642016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.16.0-77-generic2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=yunyu2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/yunyu2016-09-30 01:51:22,273 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/data/zookeeper-3.4.82016-09-30 01:51:22,275 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=127.0.0.1:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@71adff7cWelcome to ZooKeeper!2016-09-30 01:51:22,299 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server 127.0.0.1/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)2016-09-30 01:51:22,303 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@876] - Socket connection established to 127.0.0.1/127.0.0.1:2181, initiating sessionJLine support is enabled2016-09-30 01:51:22,337 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server 127.0.0.1/127.0.0.1:2181, sessionid = 0x1577a41e9b90000, negotiated timeout = 30000WATCHER::WatchedEvent state:SyncConnected type:None path:null[zk: 127.0.0.1:2181(CONNECTED) 0]


# zkShell中输入help会提示出所有的命令参数[zk: 127.0.0.1:2181(CONNECTED) 0] help
ZooKeeper host:port cmd args
        get path [watch]
        ls path [watch]
        set path data [version]
        delquota [-n|-b] path
        quit
        printwatches on|off
        create path data acl
        stat path [watch]
        listquota path
        history
        setAcl path acl
        getAcl path
        sync path
        redo cmdno
        addauth scheme auth
        delete path [version]
        deleteall path
        setquota -n|-b val path
        
# 查看znode节点[zk: 127.0.0.1:2181(CONNECTED) 0] ls
[zookeeper]

# 创建新的znode节点，关联到"my_data"
[zk: 127.0.0.1:2181(CONNECTED) 3] create /zk_test my_dataCreated /zk_test[zk: 127.0.0.1:2181(CONNECTED) 4] ls /
[zookeeper, zk_test]

# 验证/zk_test节点已经关联到"my_data"
[zk: 127.0.0.1:2181(CONNECTED) 5] get /zk_testmy_datacZxid = 0x6ctime = Mon Aug 29 20:42:40 PDT 2016mZxid = 0x6mtime = Mon Aug 29 20:42:40 PDT 2016pZxid = 0x6cversion = 0dataVersion = 0aclVersion = 0ephemeralOwner = 0x0dataLength = 7numChildren = 0

# 修改/zk_test节点的数据关联
[zk: 127.0.0.1:2181(CONNECTED) 6] set /zk_test junkcZxid = 0x6ctime = Mon Aug 29 20:42:40 PDT 2016mZxid = 0x7mtime = Mon Aug 29 20:47:08 PDT 2016pZxid = 0x6cversion = 0dataVersion = 1aclVersion = 0ephemeralOwner = 0x0dataLength = 4numChildren = 0

[zk: 127.0.0.1:2181(CONNECTED) 7] get /zk_testjunkcZxid = 0x6ctime = Mon Aug 29 20:42:40 PDT 2016mZxid = 0x7mtime = Mon Aug 29 20:47:08 PDT 2016pZxid = 0x6cversion = 0dataVersion = 1aclVersion = 0ephemeralOwner = 0x0dataLength = 4numChildren = 0

# 删除/zk_test节点
[zk: 127.0.0.1:2181(CONNECTED) 8] delete /zk_test[zk: 127.0.0.1:2181(CONNECTED) 9] ls /[zookeeper]
```

OK，Zookeeper的cluster模式的配置就大功告成了 ^_^

参考文章：

- http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html
- http://www.ibm.com/developerworks/cn/data/library/bd-zookeeper/
- https://segmentfault.com/a/1190000003994382
- http://blog.csdn.net/cruise_h/article/details/19046357