---
title: "Hive学习（三）Hive导入数据的几种方式"
date: 2016-10-18 11:59:02
tags: [Hive, HDFS]
categories: [Hadoop]
---

### Hive导入数据的几种方式

- 从本地文件系统中导入数据到Hive表
- 从HDFS中导入数据到Hive表

上面的两种方式都是使用Hive的load语句导入数据的，具体格式如下：

```
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
```

- 如果使用了LOCAL关键字，则会在本地文件系统中寻找filepath，如果filepath是相对路径，则该路径会被解释为相对于用户的当前工作目录，用户也可以指定为本地文件指定完整URI，例如：file:///data/track.log，或者直接写为/data/track.log。Load语句将会复制由filepath指定的所有文件到目标文件系统（目标文件系统由表的location属性推断得出），然后移动文件到表中。

- 如果未使用LOCAL关键字，filepath必须指的是与目标表的location文件系统相同的文件系统上的文件（例如：HDFS文件系统）。这里Load的本质实际就是一个HDFS目录下的数据文件转移到另一个HDFS目录下的操作。

当然还有其他的Hive导入数据的方式，但这里我们重点介绍这两种，其他的导入数据方式可以参考：https://www.iteblog.com/archives/949

下面我们将具体举例分析上面两种Hive导入数据的方式，下面是我们要分析的日志文件track.log的内容

```
{"logs":[{"timestamp":"1475912701768","rpid":"63146996042563584","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}
```

#### 从本地文件系统中导入数据到Hive表

```
# 查看需要导入Hive的track.log文件内容
$ cat /data/track.log{"logs":[{"timestamp":"1475912701768","rpid":"63146996042563584","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}

# 创建表test_local_table
hive> CREATE TABLE IF NOT EXISTS test_local_table(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE;

# 从本地文件系统中导入数据到Hive表
hive> load data local inpath '/data/track.log' into table test_local_table partition (dt='2016-10-18');

# 导入完成之后，查询test_local_table表中的数据
hive> select * from test_local_table;OK[{"name":"birdben.ad.click_ad","rpid":"63146996042563584","bid":"0","uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475912715001}]	info	logs	NULL	2016-10-18Time taken: 0.106 seconds, Fetched: 1 row(s)

# 在HDFS的/hive/warehouse目录中查看track.log文件，这就是我们将本地系统文件导入到Hive之后，存储在HDFS的路径
# test_hdfs.db是我们的数据库
# test_local_table是我们创建的表
# dt=2016-10-18是我们创建的Partition
$ hdfs dfs -ls /hive/warehouse/test_hdfs.db/test_local_table/dt=2016-10-18Found 1 items-rwxr-xr-x   2 yunyu supergroup        268 2016-10-17 21:19 /hive/warehouse/test_hdfs.db/test_local_table/dt=2016-10-18/track.log

# 查看文件内容
$ hdfs dfs -cat /hive/warehouse/test_hdfs.db/test_local_table/dt=2016-10-18/track.log{"logs":[{"timestamp":"1475912701768","rpid":"63146996042563584","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}
```

#### 从HDFS中导入数据到Hive表

```
# 在HDFS中查看要导入到Hive的文件（这里我们使用之前Flume收集到HDFS的track.log的日志文件）
$ hdfs dfs -ls /flume/events/birdben.ad.click_ad/201610/
Found 1 items-rw-r--r--   2 yunyu supergroup       6776 2016-10-13 06:18 /flume/events/birdben.ad.click_ad/201610/events-.1476364421957

# 创建表test_partition_table
hive> CREATE TABLE IF NOT EXISTS test_partition_table(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE;

# 从HDFS导入数据到Hive表
hive> load data inpath '/flume/events/birdben.ad.click_ad/201610/events-.1476364421957' into table test_partition_table partition (dt='2016-10-18');

# 导入完成之后，查询test_partition_table表中的数据
hive> select * from test_partition_table;OK[{"name":"birdben.ad.click_ad","rpid":"59948935480868864","bid":null,"uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475150396804}]	info	logs	NULL	2016-10-18[{"name":"birdben.ad.click_ad","rpid":"59948935480868864","bid":null,"uid":"0","did":"0","duid":"0","hbuid":null,"ua":"","device_id":"","ip":null,"server_timestamp":1475150470244}]	info	logs	NULL	2016-10-18
...
Time taken: 0.102 seconds, Fetched: 26 row(s)

# 在HDFS中再次查看源文件，此时源文件已经在此目录下不存在了，因为已经被移动到/hive/warehouse下，所以说使用load从HDFS中导入数据到Hive的方式，是将原来HDFS文件移动到Hive默认配置的数据仓库下（即:/hive/warehouse下，此目录是在hive-site.xml配置文件中配置的）
$ hdfs dfs -ls /flume/events/rp.hb.ad.view_ad/201610

# 查看Hive默认配置的数据仓库的HDFS目录下，即可找到我们导入的文件
$ hdfs dfs -ls /hive/warehouse/test_hdfs.db/test_partition_table/dt=2016-10-18Found 1 items-rwxr-xr-x   2 yunyu supergroup       6776 2016-10-13 06:18 /hive/warehouse/test_hdfs.db/test_partition_table/dt=2016-10-18/events-.1476364421957
```

原文链接：

- http://blog.zhengdong.me/2012/02/22/hive-external-table-with-partitions/
- http://www.cnblogs.com/luogankun/p/4111145.html
- http://stackoverflow.com/questions/30907657/add-partition-after-creating-table-in-hive