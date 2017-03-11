---
title: "Logstash学习（一）Logstash的webhdfs插件"
date: 2017-02-07 17:31:30
tags: [Logstash]
categories: [Log]
---

最近公司有需要查询历史日志的需求，但是之前在ES设置只保存了最近7天的日志，为了满足需求我将ES服务器的磁盘存储升级到了500G，同时延长了日志的周期至30天，但是这只是临时的解决方案，因为查询历史日志的需求仅仅是最近一个月还不够，所以为了长远考虑需要做日志的持久化存储。

### 方案选型

目前公司的架构比较简单，现在需要同时写入到HDFS中。

```
file -> logstash -> kafka -> logstash -> elasticsearch
```

这里考虑继续使用Logstash写入到HDFS中，当然也可以选择其他方案从Kafka读取数据写入HDFS的（例如：Flume，KafkaConnector等等），使用Logstash会有下面两种方案。

- 方案一
 - 优点：Logstash读取一次日志，然后双写到ES和HDFS中
 - 缺点：日志经过Logstash处理之后，无法将原始日志写入到HDFS中，只能将处理后的日志写入到HDFS中。而且Logstash挂掉之后，会影响ES和HDFS的数据的实时性。

- 方案二
 - 优点：Logstash单独写入ES和HDFS，这样其中一个Logstash挂掉之后，不会影响另一个的数据实时性。而且Logstash单独写入HDFS可以直接保留了原始日志。
 - 缺点：需要两套Logstash集群来读取Kafka中的数据，系统开销增加。

![](http://img.blog.csdn.net/20170208115731052?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里我们的需求是要在HDFS中保存原始的日志，所以选择的方案二。

### 实例

其实Logstash官方并没有提供写入HDFS的插件，但是这里官网推荐社区开发的插件logstash-output-webhdfs

logstash-output-webhdfs插件地址：

- https://github.com/logstash-plugins/logstash-output-webhdfs

#### 安装logstash-output-webhdfs插件

```
$ cd $LS_HOME
$ ./bin/logstash-plugin install logstash-output-webhdfs
Validating logstash-output-webhdfsInstalling logstash-output-webhdfsInstallation successful
```

#### Logstash配置文件

```
input {
    file {
        path => ["/home/yunyu/Downloads/rpserver.INFO"]
        type => "go"
        start_position => "beginning"
        ignore_older => 0
    }
}

output {
    stdout {
        codec => rubydebug
    }
    webhdfs {
        workers => 2
        # hdfs的namenode地址
        host => "hadoop1"
        # Hadoop的webhdfs使用的端口
        port => 50070
        # hadoop运行的用户，以这个用户的权限去写入hdfs
        user => "yunyu"
        # 按年月日建目录，按type和小时建log文件
        path => "/logstash/%{+YYYY}/%{+MM}/%{+dd}/%{type}-%{+HH}.log"
        flush_size => 1000
        # 压缩格式，可以不压缩
        # compression => "snappy"
        idle_flush_time => 10
        retry_interval => 1
    }
}
```

#### 查看HDFS文件

```
# hadoop启动步骤略过，直接查看HDFS文件目录
$ hdfs dfs -ls /logstash/2017/02/08/Found 1 items-rwxr-xr-x   2 yunyu supergroup        282 2017-02-07 19:53 /logstash/2017/02/08/go-03.log
```

可以导出或者cat输出HDFS中的日志文件查看内容，就先写到这里了。

### 遇到问题和解决方法

刚开始在使用Logstash写入HDFS的并没有遇到什么问题，然后我把收集Go，Node，PHP的日志3个Logstash进程全部配置完成并启动，一切都很正常。然后开始在线上进行部署，但是第二天发现Go的Logstash进程已经挂掉了，查看Logstash的日志发现，我发现Logstash的日志开始出现如下的错误信息。仔细观察了一下错误信息，发现

SSH登录到服务器发现Logstash报错，连接不上Hadoop的50075端口，于是查找了一下Hadoop的50075端口是什么服务使用的，发现50075是Hadoop的DataNode的http服务端口号，jps查看了一下果然没有DataNode进程，说明DataNode进程已经挂了。

```
50070 : NameNode的http服务的端口
50075 : DataNode的http服务的端口
```

然后查看DataNode的日志文件，发现如下的错误信息，DataNode的进程挂掉是因为内存不够用了，无法创建新的线程了。

```
2017-02-10 13:38:03,040 WARN org.apache.hadoop.hdfs.server.datanode.DataNode: Unexpected exception in block pool Block pool BP-316305454-10.253.43.185-1486538923200 (Datanode Uuid 3aa12dfa-9686-462c-b871-fa995534ad11) service to yy-logs-hdfs01/10.253.43.185:9000
java.lang.OutOfMemoryError: unable to create new native thread
        at java.lang.Thread.start0(Native Method)
        at java.lang.Thread.start(Thread.java:714)
        at org.apache.hadoop.hdfs.server.datanode.DataNode.recoverBlocks(DataNode.java:2529)
        at org.apache.hadoop.hdfs.server.datanode.BPOfferService.processCommandFromActive(BPOfferService.java:699)
        at org.apache.hadoop.hdfs.server.datanode.BPOfferService.processCommandFromActor(BPOfferService.java:611)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.processCommand(BPServiceActor.java:857)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.offerService(BPServiceActor.java:672)
        at org.apache.hadoop.hdfs.server.datanode.BPServiceActor.run(BPServiceActor.java:823)
        at java.lang.Thread.run(Thread.java:745)
```

既然找到是内存不足的原因，就只能从这方面进行优化了，这里我尝试修改Hadoop的启动HeapSize大小来控制内存的使用，Hadoop的HeapSize默认是1000m。可以在环境变量配置如下信息来控制Hadoop各个进程的HeapSize大小。

```
export HADOOP_NAMENODE_OPTS="-Xms512m -Xmx512m"
export HADOOP_SECONDARYNAMENODE_OPTS="-Xms256m -Xmx256m"
export HADOOP_DATANODE_OPTS="-Xms512m -Xmx512m"
export HADOOP_CLIENT_OPTS="-Xms256m -Xmx256m"
export YARN_OPTS="-Xms256m -Xmx256m"
```

控制Hadoop的启动HeapSize大小之后，我开始重启HDFS服务和YARN服务，服务启动都正常。但是当我尝试启动Logstash开始写入数据到HDFS时，Logstash报错如下。错误信息大概的意思是HDFS的文件/logstash/2017/02/10/node_proxy-05.log被锁住了，当前Logstash进程无法写入到这个文件。

```
Failed to APPEND_FILE /logstash/2017/02/10/go-03.log for DFSClient_NONMAPREDUCE_1910367742_31 on 10.253.43.185 because this file lease is currently owned by DFSClient_NONMAPREDUCE_-73615217_30 on 10.253.43.185\\n\\tat org.apache.hadoop.hdfs.server.namenode.FSNamesystem.recoverLeaseInternal
```

我有开始Google百度相关错误信息，发现HDFS为了防止文件并发写，有一个Lease（租约）的概念。实际上HDFS（及大多数分布式文件系统）不支持文件并发写，Lease是HDFS用于保证唯一写的手段。Lease可以看做是一把带时间限制的写锁，仅持有写锁的客户端可以写文件。更多Lease相关的知识请看参考文章。

既然已经知道是HDFS的Lease锁住了文件，那就去Hadoop官网找相应释放Lease的方法即可。最后我在Hadoop的官网找到了相关的命令如下：

```
recoverLease

Usage: hdfs debug recoverLease [-path <path>] [-retries <num-retries>]

COMMAND_OPTION	Description
[-path path]	HDFS path for which to recover the lease.
[-retries num-retries]	Number of times the client will retry calling recoverLease. The default number of retries is 1.
Recover the lease on the specified path. The path must reside on an HDFS filesystem. The default number of retries is 1.

# 执行recoverLease来释放文件的锁
$ hdfs debug recoverLease -path /logstash/2017/02/10/go-03.log
```

释放完HDFS的Lease之后，我继续尝试重启Logstash，发现Logstash可以开始写入数据到HDFS中。但是还是偶尔会报下面的错误信息。仔细看错误信息发现和上面的错误类似，仍然是Lease的问题。但是这里的错误只是说Lease正在被另一个进程释放中，需要等会再试。这样就说明可能是我们Logstash写入HDFS的频率过快，导致HDFS来不及释放Lease。

```
{:timestamp=>"2017-02-10T17:19:47.073000+0800", :message=>"webhdfs write caused an exception: {\"RemoteException\":{\"exception\":\"RecoveryInProgressException\",\"javaClassName\":\"org.apache.hadoop.hdfs.protocol.RecoveryInProgressException\",\"message\":\"Failed to APPEND_FILE /logstash/2017/02/10/go-03.log for DFSClient_NONMAPREDUCE_464764675_30 on 10.253.43.185 because lease recovery is in progress. Try again later.\\n\\tat org.apache.hadoop.hdfs.server.namenode.FSNamesystem.recoverLeaseInternal(FSNamesystem.java:2920)\\n\\tat org.apache.hadoop.hdfs.server.namenode.FSNamesystem.appendFileInternal(FSNamesystem.java:2685)\\n\\tat org.apache.hadoop.hdfs.server.namenode.FSNamesystem.appendFileInt(FSNamesystem.java:2985)\\n\\tat org.apache.hadoop.hdfs.server.namenode.FSNamesystem.appendFile(FSNamesystem.java:2952)\\n\\tat org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.append(NameNodeRpcServer.java:653)\\n\\tat org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.append(ClientNamenodeProtocolServerSideTranslatorPB.java:421)\\n\\tat org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)\\n\\tat org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)\\n\\tat org.apache.hadoop.ipc.RPC$Server.call(RPC.java:969)\\n\\tat org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2049)\\n\\tat org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2045)\\n\\tat java.security.AccessController.doPrivileged(Native Method)\\n\\tat javax.security.auth.Subject.doAs(Subject.java:422)\\n\\tat org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1657)\\n\\tat org.apache.hadoop.ipc.Server$Handler.run(Server.java:2043)\\n\"}}. Maybe you should increase retry_interval or reduce number of workers. Retrying...", :level=>:warn}
```

然后我尝试优化Logstash的如下配置

- flush_size : 如果event计数超出flush_size设置的值，即使未达到store_interval_in_secs，也会将数据发送到webhdfs
- idle_flush_time : 以x秒为间隔将数据发送到webhdfs
- retry_interval : 两次重试之间等待多长时间

- 提高flush_size的值，来减少访问webhdfs的频率，同时提高HDFS的写入量
- 降低idle_flush_time的值，因为提高了flush_size，所以可以适当的减少数据发送到webhdfs的时间间隔
- 提高retry_interval的值，来减少高频重试带来的额外负载

```
input {
    kafka {
        # 指定Zookeeper集群地址
        zk_connect => "zk1:2181,zk2:2181,zk3:2181"
        # 指定当前消费者的group_id，group_id不能和其他logstash消费者相同，>否则同时启动多个Logstash消费者offset会被覆盖
        group_id => "logstash_hdfs_go"
        # 指定消费的Topic
        topic_id => "log_go"
        # 指定消费的内容类型（默认是json）
        codec => "json"
    }
}

output {
    webhdfs {
        workers => 1
        host => "hadoop1"
        port => 50070
        user => "hadoop"
        path => "/logstash/%{+YYYY}/%{+MM}/%{+dd}/%{type}-%{+HH}.log"
        
        # flush_size => 500
        flush_size => 5000
        
        # idle_flush_time => 10
        idle_flush_time => 5
        
        # retry_interval => 3
        retry_interval => 3
    }
}
```

使用新的配置之后，重启Logstash已经看不到上面因为Lease未释放导致重试的异常信息了。折腾了一个下午也是不容易啊，最后写个博客记录一下处理过程，辛苦了。。

参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-outputs-webhdfs.html
- https://github.com/logstash-plugins/logstash-output-webhdfs
- http://heqin.blog.51cto.com/8931355/1796776
- http://www.cnblogs.com/ZisZ/p/3253570.html
- http://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#datanode
- http://jxy.me/2015/06/09/hdfs-data-visibility/
