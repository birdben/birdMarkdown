---
title: "Zookeeper学习（一）Zookeeper环境搭建"
date: 2016-09-01 22:18:35
tags: [Zookeeper]
categories: [Hadoop]
---

### Zookeeper安装
```
$ wget http://apache.fayea.com/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz
$ tar -zxvf zookeeper-3.4.8.tar.gz
$ cd zookeeper-3.4.8

# 修改zookeeper配置文件
$ cp conf/zoo_sample.cfg conf/zoo.cfg
```

### ZooKeeper Standalone模式

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
```

#### 启动Zookeeper服务端
```
$ ./bin/zkServer.sh start

ZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgStarting zookeeper ... STARTED
```


#### 检查Zookeeper启动状态
```
$ ./bin/zkServer.sh status

ZooKeeper JMX enabled by defaultUsing config: /software/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: standalone

# 查看zookeeper的PID
$ ps -ef | grep zookeeper
yunyu    16845  2068  0 11:06 pts/1    00:00:02 /usr/local/jdk1.7.0_79/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /software/zookeeper-3.4.8/bin/../build/classes:/software/zookeeper-3.4.8/bin/../build/lib/*.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/software/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/software/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/software/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/software/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/software/zookeeper-3.4.8/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /software/zookeeper-3.4.8/bin/../conf/zoo.cfg
yunyu    21506  2748  0 11:31 pts/1    00:00:00 grep --color=auto zookeeper

# 查看一下在监听2181端口的PID
$ lsof -i:2181
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAMEjava    16845 yunyu   19u  IPv6  50306      0t0  TCP *:2181 (LISTEN)
```

#### 启动Zookeeper Client端

```
# -server：client端连接的IP和端口号
$ ./zkCli.sh -server 127.0.0.1:2181

# Zookeeper Client端控制台会有类似如下的输出信息
Connecting to 127.0.0.1:21812016-08-29 20:35:25,389 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.8--1, built on 02/06/2016 03:18 GMT2016-08-29 20:35:25,392 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=ubuntu2016-08-29 20:35:25,393 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.7.0_792016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation2016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/software/jdk1.7.0_79/jre2016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/software/zookeeper-3.4.8/bin/../build/classes:/software/zookeeper-3.4.8/bin/../build/lib/*.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/software/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/software/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/software/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/software/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/software/zookeeper-3.4.8/bin/../conf:2016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib2016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp2016-08-29 20:35:25,396 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>2016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux2016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd642016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.16.0-77-generic2016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=yunyu2016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/yunyu2016-08-29 20:35:25,397 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/software/zookeeper-3.4.8/bin2016-08-29 20:35:25,398 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=127.0.0.1:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@56606032Welcome to ZooKeeper!2016-08-29 20:35:25,416 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server 127.0.0.1/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)2016-08-29 20:35:25,420 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@876] - Socket connection established to 127.0.0.1/127.0.0.1:2181, initiating sessionJLine support is enabled[zk: 127.0.0.1:2181(CONNECTING) 0] 2016-08-29 20:35:25,447 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server 127.0.0.1/127.0.0.1:2181, sessionid = 0x156d969c5940002, negotiated timeout = 30000WATCHER::WatchedEvent state:SyncConnected type:None path:null[zk: 127.0.0.1:2181(CONNECTED) 0][zk: 127.0.0.1:2181(CONNECTED) 0][zk: 127.0.0.1:2181(CONNECTED) 0]


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

[zk: 127.0.0.1:2181(CONNECTED) 7] get /zk_test     junkcZxid = 0x6ctime = Mon Aug 29 20:42:40 PDT 2016mZxid = 0x7mtime = Mon Aug 29 20:47:08 PDT 2016pZxid = 0x6cversion = 0dataVersion = 1aclVersion = 0ephemeralOwner = 0x0dataLength = 4numChildren = 0

# 删除/zk_test节点
[zk: 127.0.0.1:2181(CONNECTED) 8] delete /zk_test[zk: 127.0.0.1:2181(CONNECTED) 9] ls /[zookeeper]
```

#### 检查Zookeeper Client连接后的进程状态

```
# 查看zookeeper的PID
$ ps -ef | grep zookeeper
yunyu@ubuntu:/software/zookeeper-3.4.8/bin$ ps -ef | grep zookeeperyunyu    16845  2068  0 11:06 pts/1    00:00:02 /usr/local/jdk1.7.0_79/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /software/zookeeper-3.4.8/bin/../build/classes:/software/zookeeper-3.4.8/bin/../build/lib/*.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/software/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/software/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/software/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/software/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/software/zookeeper-3.4.8/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /software/zookeeper-3.4.8/bin/../conf/zoo.cfgyunyu    17569 17564  0 11:09 pts/7    00:00:02 /usr/local/jdk1.7.0_79/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /software/zookeeper-3.4.8/bin/../build/classes:/software/zookeeper-3.4.8/bin/../build/lib/*.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-log4j12-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/slf4j-api-1.6.1.jar:/software/zookeeper-3.4.8/bin/../lib/netty-3.7.0.Final.jar:/software/zookeeper-3.4.8/bin/../lib/log4j-1.2.16.jar:/software/zookeeper-3.4.8/bin/../lib/jline-0.9.94.jar:/software/zookeeper-3.4.8/bin/../zookeeper-3.4.8.jar:/software/zookeeper-3.4.8/bin/../src/java/lib/*.jar:/software/zookeeper-3.4.8/bin/../conf: org.apache.zookeeper.ZooKeeperMain -server 127.0.0.1:2181yunyu    21506  2748  0 11:31 pts/1    00:00:00 grep --color=auto zookeeper

# 查看一下在监听2181端口的PID
yunyu@ubuntu:/software/zookeeper-3.4.8/bin$ lsof -i:2181COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAMEjava    16845 yunyu   19u  IPv6  50306      0t0  TCP *:2181 (LISTEN)java    16845 yunyu   20u  IPv6  51041      0t0  TCP localhost:2181->localhost:40895 (ESTABLISHED)java    17569 yunyu   13u  IPv6  51020      0t0  TCP localhost:40895->localhost:2181 (ESTABLISHED)
```

OK，Zookeeper的standalone模式的配置就大功告成了 ^_^

参考文章：

- http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html