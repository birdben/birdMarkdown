---
title: "Kafka学习（一）Kafka环境搭建"
date: 2016-10-09 10:57:43
tags: [Kafka]
categories: [MQ]
---

### Kafka安装
```
$ wget http://apache.fayea.com/kafka/0.10.0.0/kafka_2.11-0.10.0.0.tgz
$ tar -xzf kafka_2.11-0.10.0.0.tgz
$ mv kafka_2.11-0.10.0.0 kafka_2.11
$ cd kafka_2.11
```

### 启动Kafka单节点模式

在启动Kafka之前需要先启动Zookeeper，因为Kafka集群是依赖于Zookeeper服务的。如果没有外置的Zookeeper集群服务可以使用Kafka内置的Zookeeper实例

##### 启动Kafka内置的Zookeeper
```
$ ./bin/zookeeper-server-start.sh config/zookeeper.properties
[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
...
```

这里我们使用我们自己的Zookeeper集群，所以直接启动我们搭建好的Zookeeper集群

##### 启动外置的Zookeeper集群
```
# 分别启动Hadoop1，Hadoop2，Hadoop3三台服务器的Zookeeper服务
$ ./bin/zkServer.sh startZooKeeper JMX enabled by defaultUsing config: /data/zookeeper-3.4.8/bin/../conf/zoo.cfgStarting zookeeper ... already running as process 4468.

# 分别查看一下Zookeeper服务的状态
$ ./bin/zkServer.sh statusZooKeeper JMX enabled by defaultUsing config: /data/zookeeper-3.4.8/bin/../conf/zoo.cfgMode: leader
```

##### 修改server.properties配置文件
```
# 添加外置的Zookeeper集群配置zookeeper.connect=10.10.1.64:2181,10.10.1.94:2181,10.10.1.95:2181
```

##### 启动Kafka
```
$ ./bin/kafka-server-start.sh config/server.properties
```

##### 创建Topic
```
# 创建Topic test1
$ ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test1Created topic "test1".

# 查看我们所有的Topic，可以看到test1
$ ./bin/kafka-topics.sh --list --zookeeper localhost:2181__consumer_offsetsconnect-testkafka_testmy-replicated-topicstreams-file-inputtest1

# 通过ZK的客户端连接到Zookeeper服务，localhost可以替换成Zookeeper集群的任意节点（10.10.1.64，10.10.1.94，10.10.1.95），当前localhost是10.10.1.64机器
$ ./bin/zkCli.sh -server localhost:2181

# 可以在Zookeeper中查看到新创建的Topic test1
[zk: localhost:2181(CONNECTED) 5] ls /brokers/topics[kafka_test, test1, streams-file-input, __consumer_offsets, connect-test, my-replicated-topic]
```

##### 启动producer服务，向test1的Topic中发送消息
```
$ ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test1
this is a message
this is another message
still a message
```

##### 启动consumer服务，从test1的Topic中接收消息
```
$ ./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test1 --from-beginningthis is a messagethis is another messagestill a message
```


### 启动Kafka集群模式

以上是Kafka单节点模式启动，集群模式启动只需要启动多个Kafka broker，我们这里部署了三个Kafka
broker，分别在10.10.1.64，10.10.1.94，10.10.1.95三台机器上

##### 修改server.properties配置文件
```
# 分别在10.10.1.64，10.10.1.94，10.10.1.95三台机器上的配置文件设置broker.id为0，1，2
# broker.id是用来唯一标识Kafka集群节点的
broker.id=1
```

##### 分别启动三台机器的Kafka服务
```
$ ./bin/kafka-server-start.sh config/server.properties &
```

##### 创建Topic
```
# 创建新的Topic kafka_cluster_topic
$ ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic kafka_cluster_topic

# 查看Topic kafka_cluster_topic的状态，发现Leader是1（broker.id=1）,有三个备份分别是0，1，2
$ ./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic kafka_cluster_topicTopic:kafka_cluster_topic	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: kafka_cluster_topic	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	
# 再次查看原来的Topic test1，发现Leader是0（broker.id=0）,因为我们之前单节点是在broker.id=0这台服务器（10.10.1.64）上运行的，因为当时只有这一个节点，所以leader一定是0
$ ./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test1Topic:test1	PartitionCount:1	ReplicationFactor:1	Configs:	Topic: test1	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	
# leader：是随机挑选出来的
# replicas：是负责同步leader的log的备份节点列表
# isr：是备份节点列表的子集，表示正在进行同步log的工作状态的节点列表
```

##### 启动producer服务，向kafka\_cluster\_topic的Topic中发送消息
```
$ ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic kafka_cluster_topic
this is a message
my name is birdben
```

##### 启动consumer服务，从kafka\_cluster\_topic的Topic中接收消息
```
$ ./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic kafka_cluster_topic --from-beginningthis is a message
my name is birdben
```

##### 停止leader=1的Kafka服务（10.10.1.94）
```
# 停止leader的Kafka服务之后，再次查看Topic kafka_cluster_topic的状态
# 这时候会发现Leader已经变成0了，而且Isr列表中已经没有1了，说明1的Kafka的备份服务已经停止不工作了
$ ./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic kafka_cluster_topicTopic:kafka_cluster_topic	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: kafka_cluster_topic	Partition: 0	Leader: 0	Replicas: 1,0,2	Isr: 0,2
	
# 但是此时我们仍然可以在0，2两个Kafka节点接收消息
$ ./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic kafka_cluster_topic
this is a messagebirdben
```

刚开始接触Kafka，所以只是按照官网的示例简单安装了环境，后续会随着深入使用更新复杂的配置和用法

参考文章：

- http://kafka.apache.org/quickstart