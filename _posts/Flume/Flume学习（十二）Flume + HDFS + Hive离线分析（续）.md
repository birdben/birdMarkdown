---
title: "Flume学习（十二）Flume + HDFS + Hive离线分析（续）"
date: 2016-10-12 13:43:55
tags: [Flume, HDFS, Hive]
categories: [Log]
---

上一篇中我们已经实现了使用Flume收集日志并且输出到HDFS中，并且结合Hive在HDFS进行离线的查询分析。但是也同样遇到了一些问题，本篇将解决更复杂的日志收集情况，将不同的日志格式写入到同一个日志文件，然后用Flume根据Header来写入到HDFS不同的目录。

### 日志结构

我们会讲所有的日志都写入到track.log文件中，包含API调用的日志以及其他埋点日志，这里是通过name来区分日志类型的，不同的日志类型有着不同的json结构。

```
### API日志

{"logs":[{"name":"birdben.api.call","request":"POST /api/message/receive","status":"succeeded","bid":"59885256139866115","uid":"","did":"1265","duid":"dxf536","hb_uid":"59885256030814209","ua":"Dalvik/1.6.0 (Linux; U; Android 4.4.4; YQ601 Build/KTU84P)","device_id":"fa48a076-f35f-3217-8575-5fc1f02f1ac0","ip":"::ffff:10.10.1.242","server_timestamp":1475912702996}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:02.996Z"}
{"logs":[{"name":"birdben.api.call","request":"GET /api/message/ad-detail","status":"succeeded","bid":"59885256139866115","uid":"","did":"1265","duid":"dxf536","hb_uid":"59885256030814209","ua":"Dalvik/1.6.0 (Linux; U; Android 4.4.4; YQ601 Build/KTU84P)","device_id":"fa48a076-f35f-3217-8575-5fc1f02f1ac0","ip":"::ffff:10.10.1.242","server_timestamp":1475912787476}],"level":"info","message":"logs","timestamp":"2016-10-08T07:46:27.476Z"}

### 打开App日志

{"logs":[{"timestamp":"1475914816071","rpid":"63152468644593670","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}
{"logs":[{"timestamp":"1475914827206","rpid":"63152468644593670","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840425}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.425Z"}
{"logs":[{"timestamp":"1475915077351","rpid":"63152468644593666","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090579}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.579Z"}

### 加载页面日志

{"logs":[{"timestamp":"1475914816133","rpid":"63152468644593670","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829332}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.332Z"}
{"logs":[{"timestamp":"1475914827284","rpid":"63152468644593670","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840498}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.499Z"}
{"logs":[{"timestamp":"1475915077585","rpid":"63152468644593666","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090789}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.789Z"}

### 点击链接日志

{"logs":[{"timestamp":"1475912701768","rpid":"63146996042563584","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}
{"logs":[{"timestamp":"1475913832349","rpid":"63148812297830402","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475913845544}],"level":"info","message":"logs","timestamp":"2016-10-08T08:04:05.544Z"}
{"logs":[{"timestamp":"1475915080561","rpid":"63152468644593666","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915093792}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:53.792Z"}
```

### 如何解析track.log日志文件中的日志

按照我们之前的做法，我们会使用Flume都讲日志的内容收集到HDFS上存储，但是这里的track.log日志文件中包含多种不同结构的json日志，而且这里的json数据结构是嵌套复杂对象的，我们不好在Hive上创建相应结构的表，只能创建一个大表要包含所有的日志字段，无法做到对某种日志的分析，如果像之前的做法可能无法满足我们的需求。

- 问题一：如何Hive解析这种嵌套复杂对象的json数据结构
- 问题二：如何将多种不同的日志在HDFS按类型分开存储

- 问题一解决办法：
在网上找到第三方的插件能够解析嵌套复杂对象的json数据结构，主要是替换Hive自己内嵌的Serde解析器（org.apache.hive.hcatalog.data.JsonSerDe），Github地址：https://github.com/rcongiu/Hive-JSON-Serde

- 问题二解决办法：
这里我有个想法是按照日志类型，我们可以区分我们的日志结构，根据name属性分为API日志，打开APP日志，加载页面日志，点击链接日志。但是要如何在Flume根据name属性区分开不同的日志内容，并且写入到HDFS的不同目录呢？答案就是使用Flume的Interceptor

### Hive安装Hive-JSON-Serde插件

```
# 从GitHub下载Hive-JSON-Serde
$ git clone https://github.com/rcongiu/Hive-JSON-Serde

# 编译打包Hive-JSON-Serde，打包成功之后会在json-serde/target目录生成相应的jar包
$ cd Hive-JSON-Serde
$ mvn package

# 复制打包好的jar到Hive的HIVE_AUX_JARS_PATH目录下，需要重启Hive服务，这样就不需要每次在Hive Shell中都进行add jar操作了
$ cp json-serde/target/json-serde-1.3.8-SNAPSHOT-jar-with-dependencies.jar /usr/local/hive/hcatalog/share/hcatalog/

# HIVE_AUX_JARS_PATH是在${HIVE_HOME}/conf/hive-env.sh配置文件中设置的
export HIVE_AUX_JARS_PATH=/usr/local/hive/hcatalog/share/hcatalog

# Hive Shell中创建表，如下
# 这里使用了我们刚刚引用的'org.openx.data.jsonserde.JsonSerDe'解析器
# 这样所有的日志都可以通过birdben_log_table表来查询，但是部分字段属性可能没有建表中包含进来，这样可能查出来的属性值是NULL
CREATE EXTERNAL TABLE IF NOT EXISTS birdben_log_table(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events';
```

### Flume的Interceptor

先回想一下我们是如何将日期作为参数写入到HDFS不同目录的，我们是在Flume中使用了Interceptor来将我们的name属性加入到Event的Header中，然后在Sink中通过获取Header中的name属性的值来写入到HDFS中的不同目录。

##### Flume的配置文件

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-hdfs-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 10.10.1.64
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8


#定义拦截器，为消息添加时间戳和Host地址
#将日志中的name属性添加到Header中，用来做HDFS存储的目录结构，type_name属性就是从日志文件中解析出来的name属性的值
agentX.sources.flume-avro-sink.interceptors = i1 i2
agentX.sources.flume-avro-sink.interceptors.i1.type = timestamp
agentX.sources.flume-avro-sink.interceptors.i2.type = regex_extractor
agentX.sources.flume-avro-sink.interceptors.i2.regex = "name":"(.*?)"
agentX.sources.flume-avro-sink.interceptors.i2.serializers = s1
agentX.sources.flume-avro-sink.interceptors.i2.serializers.s1.name = type_name

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

agentX.sinks.flume-hdfs-sink.type = hdfs
agentX.sinks.flume-hdfs-sink.channel = chX
agentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/%{type_name}
agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream
agentX.sinks.flume-hdfs-sink.hdfs.filePrefix = events-
agentX.sinks.flume-hdfs-sink.hdfs.rollInterval = 300
agentX.sinks.flume-hdfs-sink.hdfs.rollSize = 0
agentX.sinks.flume-hdfs-sink.hdfs.rollCount = 300
```

##### 在HDFS中查看文件目录
```
# 可以看到HDFS文件目录已经按照我们的name属性区分开了
$ hdfs dfs -ls /flume/eventsdrwxr-xr-x   - yunyu supergroup          0 2016-10-11 03:58 /flume/events/birdben.api.calldrwxr-xr-x   - yunyu supergroup          0 2016-10-11 03:58 /flume/events/birdben.ad.click_addrwxr-xr-x   - yunyu supergroup          0 2016-10-11 03:58 /flume/events/birdben.ad.open_hbdrwxr-xr-x   - yunyu supergroup          0 2016-10-11 03:58 /flume/events/birdben.ad.view_ad

$ hdfs dfs -ls /flume/events/birdben.ad.click_adFound 1 items-rwxr-xr-x   2 yunyu supergroup        798 2016-10-11 03:58 /flume/events/birdben.ad.click_ad/events-.1476183217539
```

### Hive按照不同的HDFS目录建表
```
# Hive中我们重新建表，这次我们按照HDFS已经分好的目录建表
CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_click_ad(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.click_ad';

CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_open_hb(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.open_hb';

CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_view_ad(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.view_ad';

# 在Hive中查询birdben_ad_click_ad表中的数据
hive> select * from birdben_ad_click_ad;OK[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULLTime taken: 0.519 seconds, Fetched: 3 row(s)

# 在Hive中查询birdben_ad_click_ad表中的数据总数
hive> select count(*) from birdben_ad_click_ad;Query ID = yunyu_20161011234624_fbd62672-91ee-4497-8ea1-f5a1e765a147Total jobs = 1Launching Job 1 out of 1Number of reduce tasks determined at compile time: 1In order to change the average load for a reducer (in bytes):  set hive.exec.reducers.bytes.per.reducer=<number>In order to limit the maximum number of reducers:  set hive.exec.reducers.max=<number>In order to set a constant number of reducers:  set mapreduce.job.reduces=<number>Starting Job = job_1476004456759_0008, Tracking URL = http://hadoop1:8088/proxy/application_1476004456759_0008/Kill Command = /data/hadoop-2.7.1/bin/hadoop job  -kill job_1476004456759_0008Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 12016-10-11 23:46:33,190 Stage-1 map = 0%,  reduce = 0%2016-10-11 23:46:39,554 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 0.95 sec2016-10-11 23:46:48,909 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 2.56 secMapReduce Total cumulative CPU time: 2 seconds 560 msecEnded Job = job_1476004456759_0008MapReduce Jobs Launched: Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 2.56 sec   HDFS Read: 8849 HDFS Write: 2 SUCCESSTotal MapReduce CPU Time Spent: 2 seconds 560 msecOK3Time taken: 25.73 seconds, Fetched: 1 row(s)
```

到此为止，我们上面说的两个问题都得到了解决，后续还会继续调优。

参考文章：

- http://blog.csdn.net/ahjzgyxy/article/details/44423025
- http://blog.csdn.net/xiao_jun_0820/article/details/38333171
- http://lxw1234.com/archives/2015/11/543.htm
- http://lxw1234.com/archives/2015/11/545.htm