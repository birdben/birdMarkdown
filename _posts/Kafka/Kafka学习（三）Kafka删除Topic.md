---
title: "Kafka学习（三）Kafka删除Topic"
date: 2016-11-22 02:21:01
tags: [Kafka]
categories: [MQ]
---

### 环境说明

- zookeeper-3.4.8
- kafka_2.11-0.9.0.0

最近在测试Kafka的时候创建了很多个Topic，感觉有些Topic也没什么用可以删掉了，使用Kafka的delete操作如果没有开启delete.topic.enable配置是不会删除的，而Kafka只是将Topic标识成deleted状态做逻辑删除，并且在Zookeeper中的/admin/delete_topics下创建对应的子节点。

```
$ kafka-topics.sh --delete --zookeeper localhost:2181 --topic test
Topic test is marked for deletion.Note: This will have no impact if delete.topic.enable is not set to true.
```

但是不太清楚如何完全删除掉Kafka中的Topic，后来查询并且实验了一下Kafka删除Topic的两种方式。

Kafka删除Topic的两种方式：

1. 开启Kafka的delete.topic.enable=true配置（推荐使用）
2. 手动删除Zookeeper相关数据

### 方式一（推荐使用）

- 优点：由Kafka来完成Topic的相关删除，只需要修改server.properties配置文件的delete.topic.enable为true就可以了
- 缺点：需要重启Kafka来完成配置文件的生效

##### 修改server.properties
```
# 默认是false
delete.topic.enable = true
```

##### 验证方式一
```
# 创建新的Topic logstash_test（拥有3个副本）
$ kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic logstash_test

# 查看Topic logstash_test的状态，发现Leader是1（broker.id=1）,有三个备份分别是0，1，2
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic logstash_testTopic: logstash_test	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: logstash_test	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2

# 查看Zookeeper上的Topic
$ zkCli.sh -server localhost:2181
[zk: localhost:2181(CONNECTED) 2] ls /brokers/topics [logstash_test]

[zk: localhost:2181(CONNECTED) 3] ls /config/topics[logstash_test, streams-file-input, test1, __consumer_offsets, connect-test, my-replicated-topic, kafka_test, kafka_cluster_topic]

# 查看Kafka的server.properties配置文件中log.dirs=/tmp/kafka-logs的目录
$ ll /tmp/kafka-logs/logstash_test-0total 37812drwxrwxr-x  2 yunyu yunyu     4096 Nov 21 22:38 ./drwxrwxr-x 57 yunyu yunyu     4096 Nov 22 10:58 ../-rw-rw-r--  1 yunyu yunyu 10485760 Nov 21 22:41 00000000000000000000.index-rw-rw-r--  1 yunyu yunyu 38681667 Nov 21 22:42 00000000000000000000.log

# 删除Topic logstash_test
$ kafka-topics.sh --delete --zookeeper localhost:2181 --topic logstash_test
Topic logstash_test is marked for deletion.Note: This will have no impact if delete.topic.enable is not set to true.

# 再次查看Topic logstash_test的状态，已经没有内容输出，说明Topic已经被删除了
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic logstash_test

# 再次查看Zookeeper上的Topic，logstash_test也已经被删除了
$ zkCli.sh -server localhost:2181
[zk: localhost:2181(CONNECTED) 2] ls /brokers/topics []

[zk: localhost:2181(CONNECTED) 3] ls /config/topics[streams-file-input, test1, __consumer_offsets, connect-test, my-replicated-topic, kafka_test, kafka_cluster_topic]

# 再次查看/tmp/kafka-logs目录，logstash_test相关日志也被删除了
$ ll /tmp/kafka-logs/logstash_test*
```

通过上述步骤验证，修改Kafka的delete.topic.enable配置来删除Topic十分彻底。

### 方式二

- 优点：不需要重启Kafka服务，直接删除Topic对应的系统日志，然后在Zookeeper中删除对应的目录。
- 缺点：需要人为手动删除，删除之后重新创建同名的Topic会有问题（使用方式一不会有此问题）

##### 验证方式二
```
# 创建新的Topic logstash_test（拥有3个副本）
$ kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic logstash_test

# 查看Topic logstash_test的状态，发现Leader是1（broker.id=1）,有三个备份分别是0，1，2
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic logstash_testTopic: logstash_test	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: logstash_test	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2

# 查看Zookeeper上的Topic
$ zkCli.sh -server localhost:2181
[zk: localhost:2181(CONNECTED) 2] ls /brokers/topics [logstash_test]

[zk: localhost:2181(CONNECTED) 3] ls /config/topics[logstash_test, streams-file-input, test1, __consumer_offsets, connect-test, my-replicated-topic, kafka_test, kafka_cluster_topic]

# 查看Kafka的server.properties配置文件中log.dirs=/tmp/kafka-logs的目录
$ ll /tmp/kafka-logs/logstash_test-0total 37812drwxrwxr-x  2 yunyu yunyu     4096 Nov 21 22:38 ./drwxrwxr-x 57 yunyu yunyu     4096 Nov 22 10:58 ../-rw-rw-r--  1 yunyu yunyu 10485760 Nov 21 22:41 00000000000000000000.index-rw-rw-r--  1 yunyu yunyu 38681667 Nov 21 22:42 00000000000000000000.log

# 删除Topic logstash_test的log文件（这里Kafka集群的所有节点都要删除）
$ rm -rf /tmp/kafka-logs/logstash_test*

# 删除Zookeeper上的Topic
$ zkCli.sh -server localhost:2181
[zk: localhost:2181(CONNECTED) 2] rmr /brokers/topics/logstash_test []

[zk: localhost:2181(CONNECTED) 3] rmr /config/topics/logstash_test

# 再次查看Topic logstash_test的状态，已经没有内容输出，说明Topic已经被删除了
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic logstash_test

# 貌似这种方式也能达到方式一同样的效果，但是偶然发现该方式删除之后创建同名的Topic会有问题
# 再次创建Topic logstash_test
$ kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic logstash_test

# 查看Topic logstash_test的状态，发现Leader是none，isr为空
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic logstash_testTopic:logstash_test	PartitionCount:1	ReplicationFactor:3	Configs:	Topic: logstash_test	Partition: 0	Leader: none	Replicas: 1,2,0	Isr:
```

但是重启Kafka之后创建该Topic就不会有Leader是none，isr为空的这个问题，如果方式二也需要重启Kafka就没有方式一只修改配置重启一次方便了，所以还是不建议手动删除Kafka的Topic，推荐使用Kafka官方修改配置的方式。


参考文章：

- http://blog.csdn.net/weipanp/article/details/46330471
- http://blog.csdn.net/xiaoyu_bd/article/details/52268647