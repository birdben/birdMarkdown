---
title: "Flume学习（十四）Flume整合Kafka"
date: 2016-10-17 13:32:03
tags: [Flume, Kafka]
categories: [Log]
---

### 环境简介

- JDK1.7.0_79
- Flume1.6.0
- kafka_2.11-0.9.0.0

### Flume整合Kafka的相关配置

#### flume\_agent\_file.conf配置文件
```
agentX.sources = sX
agentX.channels = chX
agentX.sinks = sk1 sk2

agentX.sources.sX.channels = chX
agentX.sources.sX.type = exec
agentX.sources.sX.command = tail -F -n +0 /Users/yunyu/Downloads/track.log

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

# Configure sinks
agentX.sinks.sk1.channel = chX
agentX.sinks.sk1.type = avro
agentX.sinks.sk1.hostname = hadoop1
agentX.sinks.sk1.port = 41414

agentX.sinks.sk2.channel = chX
agentX.sinks.sk2.type = avro
agentX.sinks.sk2.hostname = hadoop2
agentX.sinks.sk2.port = 41414

# Configure loadbalance
agentX.sinkgroups = g1
agentX.sinkgroups.g1.sinks = sk1 sk2
agentX.sinkgroups.g1.processor.type = load_balance
agentX.sinkgroups.g1.processor.backoff = true
agentX.sinkgroups.g1.processor.selector = round_robin
```

#### flume\_collector\_kafka.conf配置文件
```
agentX.sources = flume-avro-sinkagentX.channels = chXagentX.sinks = flume-kafka-sinkagentX.sources.flume-avro-sink.channels = chXagentX.sources.flume-avro-sink.type = avroagentX.sources.flume-avro-sink.bind = hadoop1agentX.sources.flume-avro-sink.port = 41414agentX.sources.flume-avro-sink.threads = 8agentX.channels.chX.type = memoryagentX.channels.chX.capacity = 10000agentX.channels.chX.transactionCapacity = 100agentX.sinks.flume-kafka-sink.type = org.apache.flume.sink.kafka.KafkaSinkagentX.sinks.flume-kafka-sink.topic = kafka_cluster_topicagentX.sinks.flume-kafka-sink.brokerList = hadoop1:9092,hadoop2:9092,hadoop3:9092agentX.sinks.flume-kafka-sink.requiredAcks = 1agentX.sinks.flume-kafka-sink.batchSize = 20agentX.sinks.flume-kafka-sink.channel = chX
```

#### 启动Flume Agent

启动Flume Agent监听track.log日志文件的变化，并且上报的Flume Collector

```
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_agent_file.conf -Dflume.root.logger=DEBUG,console -n agentX
```

#### 启动Flume Collector

启动Flume Collector监听Agent上报的消息

```
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_collector_kafka.conf -Dflume.root.logger=DEBUG,console -n agentX
```

#### 启动Kafka
```
# 启动Zookeeper服务（我这里是启动的外置Zookeeper集群，不是Kafka内置的Zookeeper）
$ ./bin zkServer.sh start

# 启动Kafka服务
$ ./bin/kafka-server-start.sh -daemon config/server.properties

# 如果是第一次启动Kafka，需要创建一个Topic，用于存储Flume收集上来的日志消息
$ ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic kafka_cluster_topic
```

#### 启动Kafka Consumer

启动Kafka Consumer来消费Kafka中的消息，这时候如果track.log日志文件有新日志写入，通过Flume上传并且写入到Kafka，最终可以在Kafka Consumer消费端看到日志文件中的内容。

```
./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic kafka_cluster_topic --from-beginning

this is a message
birdben is my name
...
```

参考文章：

- https://flume.apache.org/FlumeUserGuide.html#kafka-sink