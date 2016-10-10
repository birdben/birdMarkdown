---
title: "Flume学习（十一）Flume + HDFS + Hive离线分析"
date: 2016-10-10 16:31:12
tags: [Flume, HDFS, Hive]
categories: [Log]
---

上一篇中我们已经实现了使用Flume收集日志并且输出到HDFS中，本篇我们将结合Hive在HDFS进行离线的查询分析。具体Hive整合HDFS的环境配置请参考之前的文章。

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

#### 在HDFS中查看日志文件

```
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

#### 遇到的问题
```
# 之前我们在Flume中配置了采集到的日志输出到HDFS的保存路径是下面两种，一种使用了日期分割的，一种是没有使用日期分割的
- hdfs://10.10.1.64:8020/flume/events/20160923
- hdfs://10.10.1.64:8020/flume/events/

# 如果我们使用第二种不用日期分割的方式，在Hive上创建表指定/flume/events路径是没有问题，查询数据也都正常，但是如果使用第一种日期分割的方式，在Hive上创建表就必须指定具体的子目录，而不是/flume/events根目录，这样虽然表能够建成功但是却查询不到任何数据，因为指定的对应HDFS目录不正确，应该指定为/flume/events/20160923。这个问题确实也困扰我很久，最后才发现原来是Hive建表指定的HDFS目录不正确。

# 建议的解决方式是使用Hive的表分区来做，需要调研Hive的表分区是否支持使用HDFS已经分割好的目录结构（需要调研）
```