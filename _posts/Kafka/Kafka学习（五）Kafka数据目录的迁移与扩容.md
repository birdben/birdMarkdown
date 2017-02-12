---
title: "Kafka学习（五）Kafka数据目录的迁移与扩容"
date: 2016-12-26 16:40:47
tags: [Kafka]
categories: [MQ]
---

之前我们只有一个40G的磁盘挂在到/data0目录，Kafka的数据目录和日志目录都是写在/data0这个磁盘上的。但是很快收集上来的日志就将40G的磁盘写满了，然后我们又为Kafka集群的机器加了200G的磁盘，当时的200G磁盘是分成了2个100G的分别挂在到/data1和/data2目录上，我们也将原来40G的/data0目录下的数据全部都移动到/data1目录下，然后修改Kakfa的log.dirs配置，指定了多个Kafka数据存储目录。如下：

```
log.dirs=/data1/kafka_data,/data2/kafka_data
```

重启Kafka集群之后一切都运行正常，但是/data2目录中却没有Topic的数据，当时也觉得有些奇怪，觉得可能是因为/data1磁盘没有写满就不会写入到/data2磁盘上，而且短时间内Kafka数据应该不会增长这么快，就没有深入研究。

悲剧来了，圣诞节放假期间公司线上Kafka集群突然磁盘占用已满，后来发现是日志量突然暴增，把Kafka磁盘写满了。SSH到线上Kafka服务器发现/data1磁盘已经写满了，但是仍然没有写入到/data2磁盘，说明我之前的猜想是错的（/data1写满了才会写入到/data2）。

开始以为是log.dirs配置没有生效，我又尝试在自己本机的Kafka集群尝试了修改配置，发现配置生效了，但是和线上服务器有些不同的是，在我本机的Kafka集群指定的/data2目录下是有文件的，也有Topic数据的。

```
$ ll /data/kafka_data1total 28drwxrwxr-x  4 yunyu yunyu 4096 Dec 26 16:57 ./drwxr-xr-x 17 yunyu yunyu 4096 Dec 24 23:39 ../drwxrwxr-x  2 yunyu yunyu 4096 Dec 24 23:56 go_log-0/drwxrwxr-x  2 yunyu yunyu 4096 Dec 24 23:56 golog-0/-rw-rw-r--  1 yunyu yunyu    0 Dec 24 23:56 .lock-rw-rw-r--  1 yunyu yunyu   54 Dec 24 23:56 meta.properties-rw-rw-r--  1 yunyu yunyu   25 Dec 26 16:57 recovery-point-offset-checkpoint-rw-rw-r--  1 yunyu yunyu   25 Dec 26 16:57 replication-offset-checkpoint


$ ll /data/kafka_data2total 24drwxrwxr-x  3 yunyu yunyu 4096 Dec 26 16:57 ./drwxr-xr-x 17 yunyu yunyu 4096 Dec 24 23:39 ../-rw-rw-r--  1 yunyu yunyu    0 Dec 24 23:56 .lockdrwxrwxr-x  2 yunyu yunyu 4096 Dec 24 23:56 logstash_test-0/-rw-rw-r--  1 yunyu yunyu   54 Dec 24 23:56 meta.properties-rw-rw-r--  1 yunyu yunyu   22 Dec 26 16:57 recovery-point-offset-checkpoint-rw-rw-r--  1 yunyu yunyu   22 Dec 26 16:57 replication-offset-checkpoint
```

注意：我们本地和线上的Kafka集群都是三台机器，Node1，Node2，Node3

于是我又查看了Kafka集群的其他几个节点，发现每个节点/data2都有数据，但是Topic数量却都不一样，golog这个Topic在Kafka的所有节点都有数据，但是node_log_test这个Topic却只在Node3有，在Node2和Node3就没有，这是啥情况啊？？？

后来我仔细想了一下，是我在创建Topic的时候没有指定副本个数，默认的副本个数是1个，所以就只会在Kafka集群的一个节点上有数据。为了验证我的想法，我又查询了一下这两个Topic的详细信息。

```
# golog这个Topic是有两个副本的，所以在三台机器上都有数据存储
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic gologTopic:golog	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: golog	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 0,1,2
	

# node_log_test这个Topic是没有副本的，所以在三台机器上只有Node3一台机器上有数据存储
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_log_testTopic:node_log_test	PartitionCount:1	ReplicationFactor:1	Configs:	Topic: node_log_test	Partition: 0	Leader: 2	Replicas: 2	Isr: 2
```

根据上面的信息，我猜想有可能创建Topic的时候就已经指定存储的磁盘了（因为我们创建Topic的时候使用的之前的/data0磁盘，后来扩容的时候将/data0的磁盘数据整体迁移到/data1磁盘的），即使后来我们修改了log.dirs也只会影响新创建的Topic，对以前已经创建好的Topic是没有影响的。带着这个猜想，又仔细观察了一下线上Kafka服务器的kafka_data目录，/data1和/data2下面会有一些相同的文件，其他的文件夹都是Topic数据（log_go-0, log_node-0, log_php-0这三个是我们的Topic数据）。这几个相同的文件应该是指定log.dirs配置Kafka启动的时候创建的。

```
$ ls /data1/kafka_data
.lock
cleaner-offset-checkpoint
log_go-0/
log_node-0/
log_php-0/
meta.properties
recovery-point-offset-checkpoint
replication-offset-checkpoint


$ ls /data2/kafka_data
.lock
cleaner-offset-checkpoint
meta.properties
recovery-point-offset-checkpoint
replication-offset-checkpoint
```

发现只有recovery-point-offset-checkpoint和replication-offset-checkpoint文件不同，/data1目录下的这两个文件都有记录对应Topic和offset，而/data2目录下的这两个文件都是空的。

```
$ vi /data1/kafka_data/recovery-point-offset-checkpoint
0
3
log_node 0 18213007
log_go 0 41049987
log_php 0 100203453


$ vi /data1/kafka_data/replication-offset-checkpoint
0
3
log_node 0 18623953
log_go 0 41248972
log_php 0 100727371
```

我大胆的猜测，Kafka就是靠这个来读取数据的，我只要把磁盘占用比较大的Topic数据移动到/data2/kafka_data目录下，并且把两个文件的内容修改正确，应该就可以做到安全迁移。

按照上面的思路，我将log_php这个占用47G的Topic迁移到/data2/kafka_data目录下，然后修改对应的recovery-point-offset-checkpoint和replication-offset-checkpoint文件内容如下：

```
$ vi /data1/kafka_data/recovery-point-offset-checkpoint
0
2
log_node 0 18213007
log_go 0 41049987

$ vi /data1/kafka_data/replication-offset-checkpoint
0
1
log_php 0 100727371

$ vi /data2/kafka_data/recovery-point-offset-checkpoint
0
2
log_node 0 18213007
log_go 0 41049987


$ vi /data2/kafka_data/replication-offset-checkpoint
0
1
log_php 0 100727371
```

这里需要注意一下，第二行的数字是当前目录下的Topic数量，这一点是我根据本地Kafka集群和线上Kafka集群的配置的不同而推测出来的。

最后重启Kafka集群成功了~~

参考文章：

- http://blog.csdn.net/rongyongfeikai2/article/details/49949115