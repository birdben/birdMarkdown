---
title: "Flume学习（九）Flume整合HDFS（一）"
date: 2016-09-22 18:35:32
tags: [Flume, HDFS]
categories: [Log]
---

### 环境简介

- JDK1.7.0_79
- Flume1.6.0
- Hadoop2.7.1

之前介绍了Flume整合ES，本篇主要介绍Flume整合HDFS，将日志内容通过Flume传输给Hadoop，并且保存成文件存储在HDFS上。

### command.log日志文件

```
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
```

### Flume相关配置

#### Flume Agent端的flume\_agent\_file.conf配置

这里是采集/Users/yunyu/Downloads/command.log日志文件的内容，并且上报到127.0.0.1:41414服务器上（也就是Flume Collector端）

```
agent3.sources = command-logfile-source
agent3.channels = ch3
agent3.sinks = flume-avro-sink

agent3.sources.command-logfile-source.channels = ch3
agent3.sources.command-logfile-source.type = exec
agent3.sources.command-logfile-source.command = tail -F /Users/yunyu/Downloads/command.log

agent3.channels.ch3.type = memory
agent3.channels.ch3.capacity = 1000
agent3.channels.ch3.transactionCapacity = 100

agent3.sinks.flume-avro-sink.channel = ch3
agent3.sinks.flume-avro-sink.type = avro
agent3.sinks.flume-avro-sink.hostname = 127.0.0.1
agent3.sinks.flume-avro-sink.port = 41414
```

#### Flume Collector端的flume\_collector\_hdfs.conf配置

这里监听到127.0.0.1:41414上报的内容，并且输出到HDFS中，这里需要指定HDFS的文件路径。

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-hdfs-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 127.0.0.1
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

agentX.sinks.flume-hdfs-sink.type = hdfs
agentX.sinks.flume-hdfs-sink.channel = chX
#agentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/%y-%m-%d/%H%M/%S
agentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/
# HdfsEventSink中，hdfs.fileType默认为SequenceFile，将其改为DataStream就可以按照采集的文件原样输入到hdfs，加一行agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream
agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream
agentX.sinks.flume-hdfs-sink.hdfs.filePrefix = events-
agentX.sinks.flume-hdfs-sink.hdfs.round = true
agentX.sinks.flume-hdfs-sink.hdfs.roundValue = 10
agentX.sinks.flume-hdfs-sink.hdfs.roundUnit = minute
```

#### 启动Flume

```
# 启动Flume收集端
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_collector_hdfs.conf -Dflume.root.logger=DEBUG,console -n agentX

# 启动Flume采集端，发送数据到Collector测试
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_agent_file.conf -Dflume.root.logger=DEBUG,console -n agent3
```

这里遇到个小问题，就是Flume收集的日志文件到HDFS上查看有乱码，具体查看HDFS文件内容如下

```
$ hdfs dfs -cat /flume/events/events-.1474337184903
SEQ!org.apache.hadoop.io.LongWritable"org.apache.hadoop.io.BytesWritable�w�x0�\����WEX"Ds {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Fs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Gs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Gs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Hs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Hs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Hs {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Is {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}WEX"Is {"TIME":"2016-09-20 10:05:30","HOSTNAME":"hadoop1","LI":":0","LU":"yunyu","NU":"yunyu","CMD":"tailf command.log "}
```

解决方式：HdfsEventSink中，hdfs.fileType默认为SequenceFile，将其改为DataStream就可以按照采集的文件原样输入到hdfs，加一行agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream，如果不改就会出现HDFS文件乱码问题。

#### 在HDFS中查看日志文件

```
# 之前我们在Flume中配置了采集到的日志输出到HDFS的保存路径是hdfs://10.10.1.64:8020/flume/events/

# 查看HDFS文件存储路径
$ hdfs dfs -ls /flume/events/
Found 2 items-rw-r--r--   3 yunyu supergroup       1134 2016-09-19 23:43 /flume/events/events-.1474353822776-rw-r--r--   3 yunyu supergroup        126 2016-09-19 23:44 /flume/events/events-.1474353822777

# 查看HDFS文件内容
$ hdfs dfs -cat /flume/events/events-.1474353822776
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  543  {"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
  565  {"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
```

参考文章：

- http://blog.csdn.net/cnbird2008/article/details/18967449
- http://blog.csdn.net/lifuxiangcaohui/article/details/49949865