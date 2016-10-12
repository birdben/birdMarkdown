---
title: "Hive学习（二）使用Hive进行离线分析日志"
date: 2016-09-20 15:28:15
tags: [Hive, HDFS]
categories: [Hadoop]
---

继上一篇把Hive环境安装好之后，我们要做具体的日志分析处理，这里我们的架构是使用Flume + HDFS + Hive离线分析日志。通过Flume收集日志文件中的日志，然后存储到HDFS中，在通过Hive在HDFS之上建立数据库表，进行SQL的查询分析（其实底层是mapreduce任务）。

这里我们还是处理之前一直使用的command.log命令行日志，先来看一下具体的日志文件格式

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

参考文章：

- http://blog.csdn.net/cnbird2008/article/details/18967449
- http://blog.csdn.net/lifuxiangcaohui/article/details/49949865

### Hive中创建表

下面是具体如何在Hive中基于HDFS文件创建表的

#### 启动相关服务

```
# 启动hdfs服务
$ ./sbin/start-dfs.sh

# 启动yarn服务
$ ./sbin/start-yarn.sh

# 进入hive安装目录
$ cd /data/hive-1.2.1

# 启动metastore
$ ./bin/hive --service metastore &

# 启动hiveserver2
$ ./bin/hive --service hiveserver2 &

# 启动hive shell
$ ./bin/hive shell
hive>
hive> show databases;
OK
default
Time taken: 1.323 seconds, Fetched: 1 row(s)
```

如果看过上一篇Hive环境搭建的同学，到这里应该是一切正常的。如果启动metastore或者hiveserver2服务的时候遇到'MySQL: ERROR 1071 (42000): Specified key was too long; max key length is 767 bytes'错误，将MySQL元数据的hive数据库编码方式改成latin1就好了。

参考文章

- http://blog.csdn.net/cindy9902/article/details/6215769

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

#### 使用org.apache.hadoop.hive.contrib.serde2.RegexSerDe解析日志

```
# 确认日志写入HDFS成功之后，我们需要在Hive中创建table
# 启动hive shell
$ ./bin/hive shell

# 创建新的数据库test_hdfs
hive> create database test_hdfs;
OKTime taken: 0.205 seconds

# 使用数据库test_hdfs
hive> use test_hdfs;

# 新建表command_test_table并且使用正则表达式提取日志文件中的字段信息
# ROW FORMAT SERDE：这里使用的是正则表达式匹配
# input.regex：指定配置日志的正则表达式
# output.format.string：指定提取匹配正则表达式的字段
# LOCATION：指定HDFS文件的存储路径
hive> CREATE EXTERNAL TABLE IF NOT EXISTS command_test_table(time STRING, hostname STRING, li STRING, lu STRING, nu STRING, cmd STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
"input.regex" = '"TIME":(.*),"HOSTNAME":(.*),"LI":(.*),"LU":(.*),"NU":(.*),"CMD":(.*)',
"output.format.string" = "%1$s %2$s %3$s %4$s %5$s %6$s"
)
STORED AS TEXTFILE
LOCATION '/flume/events';

# 创建成功之后，查看表中的数据发现全都是NULL，说明正则表达式没有提取到对应的字段信息
hive> select * from command_test_table;OKNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL	NULL	NULLTime taken: 0.087 seconds, Fetched: 10 row(s)
```

这里因为我们的日志是字符串内含有json，想要通过正则表达式提取json的字段属性，通过Flume的Interceptors或者Logstash的Grok表达式很容易做到，可能是我对于Hive这块研究的还不够深入，所以没有深入去研究org.apache.hadoop.hive.contrib.serde2.RegexSerDe是否支持这种正则表达式的匹配，我又尝试了一下只用空格拆分的普通字符串日志格式。

日志格式如下

```
1 2 3
4 5 6
```

```
hive> CREATE EXTERNAL TABLE IF NOT EXISTS test_table(aa STRING, bb STRING, cc STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
"input.regex" = '([^ ]*) ([^ ]*) ([^ ]*)',
"output.format.string" = "%1$s %2$s %3$s"
)
STORED AS TEXTFILE
LOCATION '/flume/events';

hive> select * from test_table;
OK
1	2	3
4	5	6Time taken: 0.035 seconds, Fetched: 2 row(s)
```

发现用这种方式能够用正则表达式解析出来我们需要提取的字段信息。不知道是不是org.apache.hadoop.hive.contrib.serde2.RegexSerDe不支持这种带有json字符串的正则表达式匹配方式。这里我换了另一种做法，修改我们的日志格式尝试一下，我把command.log的日志内容修改成纯json字符串，然后使用org.apache.hive.hcatalog.data.JsonSerDe解析json字符串的匹配。下面是修改后的command.log日志文件内容。

#### command.log日志文件

```
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 15:04:43","HOSTNAME":"localhost","LI":"806","LU":"yunyu","NU":"yunyu","CMD":"ll"}
{"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
{"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
{"TIME":"2016-09-06 13:10:43","HOSTNAME":"localhost","LI":"783","LU":"yunyu","NU":"yunyu","CMD":"ssh yunyu@10.10.1.15"}
```

#### 使用org.apache.hive.hcatalog.data.JsonSerDe解析日志

```
# Flume重新写入新的command.log日志到HDFS中
# 启动hive shell
$ ./bin/hive shell

# 使用数据库test_hdfs
hive> use test_hdfs;

# 新建表command_json_table并且使用json解析器提取日志文件中的字段信息
# ROW FORMAT SERDE：这里使用的是json解析器匹配
# LOCATION：指定HDFS文件的存储路径
hive> CREATE EXTERNAL TABLE IF NOT EXISTS command_json_table(time STRING, hostname STRING, li STRING, lu STRING, nu STRING, cmd STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events';

# 这创建还是会报错，查看hive.log日志文件的错误信息，发现是缺少org.apache.hive.hcatalog.data.JsonSerDe类所在的jar包
Caused by: java.lang.ClassNotFoundException: Class org.apache.hive.hcatalog.data.JsonSerDe not found

# 查了下Hive的官网wiki，发现需要先执行add jar操作，将hive-hcatalog-core.jar添加到classpath（具体的jar包地址根据自己实际的Hive安装路径修改）
add jar /usr/local/hive/hcatalog/share/hcatalog/hive-hcatalog-core-1.2.1.jar;

# 为了避免每次启动hive shell都重新执行一下add jar操作，我们这里在${HIVE_HOME}/conf/hive-env.sh启动脚本中添加如下信息
export HIVE_AUX_JARS_PATH=/usr/local/hive/hcatalog/share/hcatalog

# 重启Hive服务之后，再次创建command_json_table表成功
hive> CREATE EXTERNAL TABLE IF NOT EXISTS command_json_table(time STRING, hostname STRING, li STRING, lu STRING, nu STRING, cmd STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events';

# 查看command_json_table表中的内容，json字段成功的解析出我们要的字段
hive> select * from command_json_table;
OK2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 15:04:43	localhost	806	yunyu	yunyu	ll2016-09-06 13:10:43	localhost	783	yunyu	yunyu	ssh yunyu@10.10.1.152016-09-06 13:10:43	localhost	783	yunyu	yunyu	ssh yunyu@10.10.1.152016-09-06 13:10:43	localhost	783	yunyu	yunyu	ssh yunyu@10.10.1.15Time taken: 0.09 seconds, Fetched: 10 row(s)
```

参考文章：

- https://cwiki.apache.org/confluence/display/Hive/Json+SerDe
- https://my.oschina.net/cjun/blog/494692
- http://blog.csdn.net/bluishglc/article/details/46005269
- http://blog.sina.com.cn/s/blog_604c7cdd0102wbzz.html
- http://blog.csdn.net/xiao_jun_0820/article/details/38119123

#### 使用select count(*)验证Hive可以调用MapReduce进行离线任务处理

```
# 使用数据库test_hdfs
hive> use test_hdfs;

# 统计command_json_table表的行数，执行失败
hive> select count(*) from command_json_table;

# 查看yarn的log发现执行对应的mapreduce提示Connection Refused
# 因为Hive最终是调用Hadoop的MapReduce来执行任务的，所以需要查看的是yarn的log日志
appattempt_1474251946149_0003_000002. Got exception: java.net.ConnectException: Call From ubuntu/127.0.1.1 to ubuntu:50060 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused
```

这里我自己分析了一下原因，我们之前搭建的Hadoop集群配置是

```
Hadoop1节点是namenode
Hadoop2和Hadoop3这两个节点是datanode
```

仔细看了一下报错的信息，我们现在在Hadoop1上安装的Hive，ubuntu:50060这个发现是连接的Hadoop1节点的50060端口，但是50060端口是NodeManager服务的端口，但这里Hadoop1不是datanode所以没有启动NodeManager服务，需要在slaves文件中把Hadoop1节点添加上

```
# 修改好之后重启dfs和yarn服务，再次执行sql语句
hive> select count(*) from command_json_table;

# 又报如下的错误
Application application_1474265561006_0002 failed 2 times due to Error launching appattempt_1474265561006_0002_000002. Got exception: java.net.ConnectException: Call From ubuntu/127.0.1.1 to ubuntu:52990 failed on connection exception: java.net.ConnectException: Connection refused; For more details see: http://wiki.apache.org/hadoop/ConnectionRefused
```

这个问题可把我坑惨了，后来自己分析了一下，原因一定是哪里的配置是我配置错了hostname是ubuntu了，但是找了一圈的配置文件也没找到，后来看网上说在namenode节点上用yarn node -list -all查看不健康的节点，发现没有问题。又尝试hdfs dfsadmin -report语句检查 DataNode 是否正常启动，让我查出来我的/etc/hosts默认配置带有'127.0.0.1 ubuntu'，这样Hadoop可能会用ubuntu这个hostname

重试之后还是不对，使用hostname命令查看ubuntu系统的hostname果然是'ubuntu'，ubuntu系统永久修改hostname是在/etc/hostname文件中修改，我这里对应修改成Hadoop1,hadoop2,hadoop3

修改/etc/hostname文件后，重新检查Hadoop集群的所有主机的hostname都已经不再是ubuntu了，都改成对应的hadoop1，hadoop2，hadoop3

```
$ hdfs dfsadmin -report Configured Capacity: 198290427904 (184.67 GB)Present Capacity: 159338950656 (148.40 GB)DFS Remaining: 159084933120 (148.16 GB)DFS Used: 254017536 (242.25 MB)DFS Used%: 0.16%Under replicated blocks: 8Blocks with corrupt replicas: 0Missing blocks: 0Missing blocks (with replication factor 1): 0-------------------------------------------------Live datanodes (3):Name: 10.10.1.94:50010 (hadoop2)Hostname: hadoop2Decommission Status : NormalConfigured Capacity: 66449108992 (61.89 GB)DFS Used: 84217856 (80.32 MB)Non DFS Used: 8056225792 (7.50 GB)DFS Remaining: 58308665344 (54.30 GB)DFS Used%: 0.13%DFS Remaining%: 87.75%Configured Cache Capacity: 0 (0 B)Cache Used: 0 (0 B)Cache Remaining: 0 (0 B)Cache Used%: 100.00%Cache Remaining%: 0.00%Xceivers: 1Last contact: Tue Sep 20 02:23:19 PDT 2016Name: 10.10.1.64:50010 (hadoop1)Hostname: hadoop1Decommission Status : NormalConfigured Capacity: 65392209920 (60.90 GB)DFS Used: 84488192 (80.57 MB)Non DFS Used: 22853742592 (21.28 GB)DFS Remaining: 42453979136 (39.54 GB)DFS Used%: 0.13%DFS Remaining%: 64.92%Configured Cache Capacity: 0 (0 B)Cache Used: 0 (0 B)Cache Remaining: 0 (0 B)Cache Used%: 100.00%Cache Remaining%: 0.00%Xceivers: 1Last contact: Tue Sep 20 02:23:18 PDT 2016Name: 10.10.1.95:50010 (hadoop3)Hostname: hadoop3Decommission Status : NormalConfigured Capacity: 66449108992 (61.89 GB)DFS Used: 85311488 (81.36 MB)Non DFS Used: 8041508864 (7.49 GB)DFS Remaining: 58322288640 (54.32 GB)DFS Used%: 0.13%DFS Remaining%: 87.77%Configured Cache Capacity: 0 (0 B)Cache Used: 0 (0 B)Cache Remaining: 0 (0 B)Cache Used%: 100.00%Cache Remaining%: 0.00%Xceivers: 1Last contact: Tue Sep 20 02:23:20 PDT 2016
```

重启系统之后，检查hostname都已经修改正确，再次启动dfs，yarn，hive服务，重试执行select count(*) from command_json_table;终于正确了。。。

```
hive> select count(*) from command_json_table;
Query ID = yunyu_20160920020204_544583fc-b872-44c8-95a6-a7b0c9611da7Total jobs = 1Launching Job 1 out of 1Number of reduce tasks determined at compile time: 1In order to change the average load for a reducer (in bytes):  set hive.exec.reducers.bytes.per.reducer=<number>In order to limit the maximum number of reducers:  set hive.exec.reducers.max=<number>In order to set a constant number of reducers:  set mapreduce.job.reduces=<number>Starting Job = job_1474274066864_0003, Tracking URL = http://hadoop1:8088/proxy/application_1474274066864_0003/Kill Command = /data/hadoop-2.7.1/bin/hadoop job  -kill job_1474274066864_0003Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 12016-09-20 02:02:13,090 Stage-1 map = 0%,  reduce = 0%2016-09-20 02:02:19,318 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.14 sec2016-09-20 02:02:26,575 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 2.51 secMapReduce Total cumulative CPU time: 2 seconds 510 msecEnded Job = job_1474274066864_0003MapReduce Jobs Launched: Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 2.51 sec   HDFS Read: 8187 HDFS Write: 3 SUCCESSTotal MapReduce CPU Time Spent: 2 seconds 510 msecOK10Time taken: 23.155 seconds, Fetched: 1 row(s)
```

参考文章：

- http://www.powerxing.com/install-hadoop-cluster/
- http://www.th7.cn/Program/java/201609/968295.shtml
- http://blog.csdn.net/ruglcc/article/details/7802077