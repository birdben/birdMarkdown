---
title: "Hive学习（四）Hive内部表和外部表"
date: 2016-10-18 14:39:56
tags: [Hive, HDFS]
categories: [Hadoop]
---

上一篇我们介绍了Hive导入数据的两种方式，本篇我们对Hive的表进行重点介绍。上一篇我们使用的都是Hive的内部表，如何区分Hive的内部表和外部表呢？create （external） table语句是否带有external关键字，如果带有external关键字就是外部表，所以上一篇我们导入的数据都是导入到Hive的内部表，也就是文件都存储在/hive/warehouse的HDFS目录中，即Hive默认配置的数据仓库。External Table允许我们将文件保存在任意的HDFS目录下，下面将详细介绍内部表和外部表的区别。

### Hive内部表
```
# 创建内部表test_internal_table，这里创建好的表的数据文件是默认存储在/hive/warehouse目录下，全路径是/hive/warehouse/test_hdfs.db/test_internal_table
# test_hdfs是我们的数据库
# 如果删除test_internal_table，元数据表结构和数据文件都将会被删除
CREATE TABLE IF NOT EXISTS test_internal_table(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE;
```

### Hive外部表
```
# 创建外部表test_external_table，这里创建好的表是读取的Location属性指定文件目录下的数据文件，而不是默认的/hive/warehouse下，这样我们就可以使用External Table结合外部的Application使用（这里读取的是Flume采集并写入HDFS的数据文件），Hive同样可以读取Hive默认配置的数据仓库之外的HDFS目录下的数据文件。
# Location是指定的数据文件路径
# 如果删除test_external_table，元数据表结构会被删除，但是数据文件不会被删除
CREATE EXTERNAL TABLE IF NOT EXISTS test_external_table(logs array<struct<name:string, rpid:string, bid:string, uid:string, did:string, duid:string, hbuid:string, ua:string, device_id:string, ip:string, server_timestamp:BIGINT>>, level STRING, message STRING, client_timestamp BIGINT)
partitioned by (dt string)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS TEXTFILE
LOCATION '/flume/events/birdben.ad.view_ad';
```

最后总结一下Hive内部表与外部表的区别：

- 在导入数据时，导入到内部表，数据文件是存储在Hive的默认的数据仓库下的。导入到外部表，数据文件是存储在External Table指定的Location目录下的。
- 在删除内部表时，Hive将会把属于表的元数据和数据全部删掉；而删除外部表的时，Hive仅仅删除外部表的元数据，数据是不会删除的。

如何选择使用哪种表呢？

- 如果所有的数据处理都需要由Hive完成，那么建议你应该使用内部表，如果所有的数据处理需要整合其他Application一起应用（例如：Flume负责采集数据文件，并且根据Header写入到HDFS的不同目录下的数据文件），此时建议使用外部表。


原文链接：

- http://blog.csdn.net/yeruby/article/details/23033273
- http://www.aboutyun.com/thread-7458-1-1.html