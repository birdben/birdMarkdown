---
title: "Elasticsearch学习（一）集群red状态的处理"
date: 2016-12-22 21:25:14
tags: [Elasticsearch]
categories: [Search]
---

今天刚刚搭建好公司的日志收集系统，晚上的时候根据Kafka的生产和消费速度情况适当的调节了一下Logstash和ES的配置，做了一些配置的优化。但是没过多久居然出现了集群red状态的情况。突然感觉好慌张，通过head插件看了一下当前集群的问题，发现有两个索引的shared是无法分配的。

```
# 查看集群的索引情况
curl -XGET 'http://localhost:9200/_cluster/health?level=indices&pretty'

# 查看集群的分片情况
curl -XGET 'http://localhost:9200/_cluster/health?level=shards&pretty'
```
根据ES的集群API查看一下索引和分片的健康情况
以前遇到这种情况通常手动做reroute操作就可以分配了，但是这次的情况有些特殊。

```
curl -XPOST 'localhost:9200/_cluster/reroute?pretty' -d '{
    "commands" : [ {
        "allocate" : {
            "index" : "node_access_logs_index_2016.12.21",
            "shard" : 0,
            "node" : "node-1",
            "allow_primary" : true
        }
    }]
}'
```
尝试了reroute操作后，报错如下

```
{
  "state" : "INITIALIZING",
  "primary" : true,
  "node" : "MsryV3-fTOCdgolwpxd0_w",
  "relocating_node" : null,
  "shard" : 0,
  "index" : "node_track_logs_index_2016.12.21",
  "version" : 3,
  "allocation_id" : {
    "id" : "lOUerCpGQTWcX0grHWLr0Q"
  },
  "unassigned_info" : {
    "reason" : "INDEX_CREATED",
    "at" : "2016-12-22T13:03:08.686Z",
    "details" : "force allocation from previous reason ALLOCATION_FAILED, failed to create shard, failure ElasticsearchException[failed to create shard]; nested: ElasticsearchParseException[Failed to parse setting [index.refresh_interval] with value [10] as a time value: unit is missing or unrecognized]; "
  }
}
```

通过上面的错误信息，我们能够看出来其实这个问题是我马虎造成的，配置我们的Logstash创建索引的Template模板参数错误没有带时间单位，正确的配置应该是"index.refresh_interval": "10s"，我之前的配置少了单位秒，所以ES重新分片的时候会报错[Failed to parse setting [index.refresh_interval] with value [10] as a time value: unit is missing or unrecognized]

知道问题的原因之后，我尝试直接修改线上索引Mapping的setting设置，将"index.refresh_interval": "10s"的单位加上，发现报错如下，提示是当前索引的primary shared不可用，所以无法修改Mapping。

```
curl -XPOST 'localhost:9200/node_track_logs_index_2016.12.21/_settings?pretty' -d'
{
      "index.number_of_replicas": "1",
      "index.number_of_shards": "5",
      "index.refresh_interval": "10s"
}'
{
  "error" : {
    "root_cause" : [ {
      "type" : "unavailable_shards_exception",
      "reason" : "[node_track_logs_index_2016.12.21][0] primary shard is not active Timeout: [1m], request: [index {[node_track_logs_index_2016.12.21]
[_settings][AVkmqvF1jCuNyx9kvqWT], source[\n{\n      \"index.number_of_replicas\": \"1\",\n      \"index.number_of_shards\": \"5\",\n      \"index.refresh_interval\": \"10s\"\n}]}]"
    } ],
    "type" : "unavailable_shards_exception",
    "reason" : "[node_track_logs_index_2016.12.21][0] primary shard is not active Timeout: [1m], request: [index {[node_track_logs_index_2016.12.21][_settings][AVkmqvF1jCuNyx9kvqWT], source[\n{\n      \"index.number_of_replicas\": \"1\",\n      \"index.number_of_shards\": \"5\",\n      \"index.refresh_interval\": \"10s\"\n}]}]"
  },
  "status" : 503
}
```

后来仔细想了一下我们这个Template是在Logstash按天创建索引的的时候才会使用的，所以我们unassign的node_track_logs_index_2016.12.21和node_proxy_logs_index_2016.12.21索引其实是正好需要按天创建新索引创建的，所以直接删除掉该索引，重启Logstash按照新的Template创建索引的Mapping就好了。



参考文章：

- https://www.elastic.co/guide/en/elasticsearch/guide/current/_cluster_health.html#_drilling_deeper_finding_problematic_indices
- http://www.searchtech.pro/elasticsearch-manual-allocate-shard
- http://openskill.cn/article/107