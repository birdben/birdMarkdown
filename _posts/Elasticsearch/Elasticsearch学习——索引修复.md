---
title: "Elasticsearch学习——索引修复"
date: 2017-07-04 18:06:16
tags: [Elasticsearch]
categories: [Search]
---

之前公司的日志收集平台运行了一段时间后，出现了索引分片无法自动分配的问题，然后尝试手动分配如下：

```
$ curl -XPOST 'localhost:9200/_cluster/reroute' -d '{"commands":[{"allocate":{"index":"nginx_access_logs_index_2017.04.09","shard":"2","node":"node-1","allow_primary":true}}]}'

$ curl -XGET 'http://localhost:9200/_cluster/allocation/explain' -d '{"index":"nginx_access_logs_index_2017.04.09","shard":2,"primary": false}'
```

但是ES的日志会报错如下：

```
failed recovery, failure IndexShardRecoveryException[failed to fetch index version after copying it over]; nested: CorruptIndexException[i_o_exception: failed engine (reason: [merge failed]) (resource=preexisting_corruption)]; nested: NotSerializableExceptionWrapper[i_o_exception: failed engine (reason: [merge failed])]; nested: CorruptIndexException[checksum failed (hardware problem?) : expected=8d75035 actual=e8a64fa5 (resource=BufferedChecksumIndexInput(NIOFSIndexInput(path="/data0/es_data/yunyu-cluster/nodes/0/indices/nginx_access_logs_index_2017.04.09/2/index/_6f6.cfs") [slice=_6f6_Lucene54_0.dvd]))]; 
```

该错误提示Lucene的索引文件有问题，可能是磁盘问题。然后我尝试使用Lucene的CheckIndex工具修复索引，但是仍然是同样的错误。

```
$ java -cp lucene-core-5.5.0.jar -ea:org.apache.lucene... org.apache.lucene.index.CheckIndex /data0/es_data/yunyu-cluster/nodes/0/indices/nginx_access_logs_index_2017.04.09/2/index -fix

$ java -cp lucene-core-5.5.0.jar -ea:org.apache.lucene... org.apache.lucene.index.CheckIndex /data0/es_data/yunyu-cluster/nodes/0/indices/nginx_access_logs_index_2017.04.09/2/index -exorcise
```

最后实在没有别的办法只能从snapshot恢复备份数据了。

当时的日志信息没有保留完全，现在事后回忆实在是有点想不起来了，以后总结的时候要注意保留现场。

参考文章：

- https://lucene.apache.org/core/5_3_0/core/org/apache/lucene/codecs/lucene53/package-summary.html#package_description
- http://blog.csdn.net/laigood/article/details/8296678
- http://codepub.cn/2016/06/24/Lucene-6-0-in-action-the-index-of-hot-backup-and-recovery/