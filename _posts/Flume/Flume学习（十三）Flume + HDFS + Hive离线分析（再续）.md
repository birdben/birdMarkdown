---
title: "Flume学习（十三）Flume + HDFS + Hive离线分析（再续）"
date: 2016-10-13 16:52:54
tags: [Flume, HDFS, Hive]
categories: [Log]
---

在《Flume学习（十一）Flume + HDFS + Hive离线分析》这篇中我们就遇到了Hive分区的问题，这里我们再来回顾一下之前待调研的问题

```
# 问题二：
之前我们在Flume中配置了采集到的日志输出到HDFS的保存路径是下面两种，一种使用了日期分割的，一种是没有使用日期分割的
- hdfs://10.10.1.64:8020/flume/events/20160923
- hdfs://10.10.1.64:8020/flume/events/

# 解决方案：
如果我们使用第二种不用日期分割的方式，在Hive上创建表指定/flume/events路径是没有问题，查询数据也都正常，但是如果使用第一种日期分割的方式，在Hive上创建表就必须指定具体的子目录，而不是/flume/events根目录，这样虽然表能够建成功但是却查询不到任何数据，因为指定的对应HDFS目录不正确，应该指定为/flume/events/20160923。这个问题确实也困扰我很久，最后才发现原来是Hive建表指定的HDFS目录不正确。

指定location为'/flume/events'不好用，Hive中查询command_json_table表中没有数据
hive> CREATE EXTERNAL TABLE IF NOT EXISTS command_json_table(time STRING, hostname STRING, li STRING, lu STRING, nu STRING, cmd STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events';

指定location为'/flume/events/20160923'好用，Hive中查询command_json_table_20160923表中有数据
hive> CREATE EXTERNAL TABLE IF NOT EXISTS command_json_table_20160923(time STRING, hostname STRING, li STRING, lu STRING, nu STRING, cmd STRING)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/20160923';

建议的解决方式是使用Hive的表分区来做，需要调研Hive的表分区是否支持使用HDFS已经分割好的目录结构（需要调研）
```

上面是我们之前的问题原文描述，之前需要调研Hive表分区是否可以使用HDFS已经分割好的目录结构，这里我找到了一篇blog，终于理解了Hive关于External表如何使用partition的，下面给出了原文和译文的链接地址

原文链接：

- http://blog.zhengdong.me/2012/02/22/hive-external-table-with-partitions/

译文链接：

- 

我们带着上面的问题继续优化，之前的解决办法是按照我们日志中的name属性值存储在HDFS的不同目录中，本篇我们使用Partition来解决数据量增长的情况，我们在之前使用name属性的基础上在新建dt目录（按照月份来分割数据）

```
agentX.sources = flume-avro-sinkagentX.channels = chXagentX.sinks = flume-hdfs-sinkagentX.sources.flume-avro-sink.channels = chXagentX.sources.flume-avro-sink.type = avroagentX.sources.flume-avro-sink.bind = hadoop1agentX.sources.flume-avro-sink.port = 41414agentX.sources.flume-avro-sink.threads = 8#定义拦截器，为消息添加时间戳和Host地址
#将日志中的name属性添加到Header中，用来做HDFS存储的目录结构，type_name属性就是从日志文件中解析出来的name属性的值，这里使用%Y%m表达式代表按照年月分区agentX.sources.flume-avro-sink.interceptors = i1 i2agentX.sources.flume-avro-sink.interceptors.i1.type = timestampagentX.sources.flume-avro-sink.interceptors.i2.type = regex_extractoragentX.sources.flume-avro-sink.interceptors.i2.regex = "name":"(.*?)"agentX.sources.flume-avro-sink.interceptors.i2.serializers = s1agentX.sources.flume-avro-sink.interceptors.i2.serializers.s1.name = type_nameagentX.channels.chX.type = memoryagentX.channels.chX.capacity = 1000agentX.channels.chX.transactionCapacity = 100agentX.sinks.flume-hdfs-sink.type = hdfsagentX.sinks.flume-hdfs-sink.channel = chXagentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/%{type_name}/%Y%magentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStreamagentX.sinks.flume-hdfs-sink.hdfs.filePrefix = events-agentX.sinks.flume-hdfs-sink.hdfs.rollInterval = 300agentX.sinks.flume-hdfs-sink.hdfs.rollSize = 0agentX.sinks.flume-hdfs-sink.hdfs.rollCount = 300
```

##### 在HDFS中查看文件目录
```
# 可以看到HDFS文件目录已经按照我们的name属性区分开了
hdfs dfs -ls /flume/events/drwxr-xr-x   - yunyu supergroup          0 2016-10-13 07:01 /flume/events/birdben.api.calldrwxr-xr-x   - yunyu supergroup          0 2016-10-13 07:02 /flume/events/birdben.ad.click_addrwxr-xr-x   - yunyu supergroup          0 2016-10-13 07:02 /flume/events/birdben.ad.open_hbdrwxr-xr-x   - yunyu supergroup          0 2016-10-13 07:02 /flume/events/birdben.ad.view_ad

# 查看个不同name下的目录是按照年月分割开的
$ hdfs dfs -ls /flume/events/birdben.ad.click_adFound 2 itemsdrwxr-xr-x   - yunyu supergroup          0 2016-10-13 06:18 /flume/events/birdben.ad.click_ad/201610drwxr-xr-x   - yunyu supergroup          0 2016-10-13 07:07 /flume/events/birdben.ad.click_ad/201611

# 数据文件是存储在具体的年月目录下的
$ hdfs dfs -ls /flume/events/birdben.ad.click_ad/201610/Found 1 items-rw-r--r--   2 yunyu supergroup       1596 2016-10-13 06:18 /flume/events/birdben.ad.click_ad/201610/events-.1476364422107
```

### Hive按照不同的HDFS目录建表
```
# 这里我们是需要先理解Hive的内部表和外部表的区别，然后我们在之前的建表语句中加入partition分区，我们这里使用的是dt字段作为partition，dt字段不能够与建表语句中的字段重复，否则建表时会报错。
CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_click_ad(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.click_ad';

CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_open_hb(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.open_hb';

CREATE EXTERNAL TABLE IF NOT EXISTS birdben_ad_view_ad(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.view_ad';

# 这时候我们查询表，表中是没有数据的。我们需要手工添加partition分区之后，才能查到数据。
hive> select * from birdben_ad_click_ad;

# 建表完成之后，我们需要手工添加partition目录为我们Flume之前划分的好的年月目录
alter table birdben_ad_click_ad add partition(dt='201610') location '/flume/events/birdben_ad_click_ad/201610';
alter table birdben_ad_click_ad add partition(dt='201611') location '/flume/events/birdben_ad_click_ad/201611';

alter table birdben_ad_open_hb add partition(dt='201610') location '/flume/events/birdben.ad.open_hb/201610';
alter table birdben_ad_open_hb add partition(dt='201611') location '/flume/events/birdben.ad.open_hb/201611';

alter table birdben_ad_view_ad add partition(dt='201610') location '/flume/events/birdben.ad.view_ad/201610';
alter table birdben_ad_view_ad add partition(dt='201611') location '/flume/events/birdben.ad.view_ad/201611';

# 这时候我们查询表，能够查询到全部的数据了（包括201610和201611的数据）
hive> select * from birdben_ad_click_ad;
OK[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201611[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201611[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201611Time taken: 0.1 seconds, Fetched: 9 row(s)

# 也可以按照分区字段查询数据，这样就能够证明我们可以使用Hive的External表partition对应到我们Flume中创建好的 %Y%m（年月） 目录结构
hive> select * from birdben_ad_click_ad where dt = '201610';
OK[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201610[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201610Time taken: 0.099 seconds, Fetched: 6 row(s)

hive> select * from birdben_ad_click_ad where dt = '201611';
OK
[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	201611[{"name":"birdben.ad.click_ad","rpid":"63148812297830402","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475913845544}]	info	logs	NULL	201611[{"name":"birdben.ad.click_ad","rpid":"63152468644593666","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475915093792}]	info	logs	NULL	201611Time taken: 0.11 seconds, Fetched: 3 row(s)
```

### 总结

其实我写了这么多篇Flume + HDFS + Hive的文章，就是为了证明Flume可以按照指定的Header的key分别写入不同的HDFS目录，Hive又可以通过External表将Location定位到Flume写入的HDFS目录，而且还可以通过Partition分区定位到Flume设置的Header对应的目录，这样就能够比较优雅的将Flume, HDFS, Hive整合到一起了。但是还是有些需要优化的地方，比如说我们的日志格式不够规范，每种日志都有不同的格式，而且还都写入到同一个track.log日志文件中，只能通过name属性作区分。还有就是Hive的Partition每次需要手工去修改表，否则无法查询到HDFS对应目录下的数据，也有人使用 [script](https://github.com/don9z/hadoop-tools/blob/master/hive/addpartition.py) 脚本来做这些事情，待以后有时间继续深入研究。

参考文章：

- http://blog.zhengdong.me/2012/02/22/hive-external-table-with-partitions/