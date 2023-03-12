---
title: "ElasticSearch官网翻译__routingField"
date: 2017-05-13 00:11:47
tags: [Elasticsearch]
categories: [Search]
---

#### _routing field

使用以下公式将文档路由到索引中的特定分片：

```
shard_num = hash（_routing）％num_primary_shards
```

<b>用于_routing的默认值是文档的_id或文档的_parent ID（如果存在）。</b>

可以通过为每个文档指定自定义路由值来实现自定义路由模式。 例如：

```
PUT my_index/my_type/1?routing=user1 (1)
{
  "title": "This is a document"
}

GET my_index/my_type/1?routing=user1 (2)
```

- (1) 本文档使用user1作为其路由值，而不是其ID。
- (2) <b>在获取，删除或更新文档时需要提供相同的路由值。</b>

_routing域的值可以在查询，聚合，脚本以及排序时访问：

```
GET my_index/_search
{
  "query": {
    "terms": {
      "_routing": [ "user1" ] 
    }
  },
  "aggs": {
    "Routing values": {
      "terms": {
        "field": "_routing", 
        "size": 10
      }
    }
  },
  "sort": [
    {
      "_routing": { 
        "order": "desc"
      }
    }
  ],
  "script_fields": {
    "Routing value": {
      "script": "doc['_routing']" 
    }
  }
}
```

- (1) 在_routing字段上查询（也可以参见ids查询）
- (2) 在_routing字段上聚合
- (3) 在_routing字段上排序
- (4) 访问脚本中的_routing字段（必须启用inline scripts才能使此示例工作）

##### 使用自定义路由搜索

自定义路由可以减少搜索的影响。 索引中的所有碎片不必将搜索请求散开，而是将该请求发送到与特定路由值（或值）匹配的分片：

```
GET my_index/_search?routing=user1,user2 
{
  "query": {
    "match": {
      "title": "document"
    }
  }
}
```

- (1) 该搜索请求只能在与user1和user2路由值相关联的分片上执行。

##### 路由值是必须的

<b>
当使用自定义路由时，重要的是在索引，获取，删除或更新文档时提供路由值。

忘记路由值可能导致文档被索引在多个分片上。 作为保护措施，可以将_routing字段配置为使所有CRUD操作都需要自定义路由值：
</b>

```
PUT my_index
{
  "mappings": {
    "my_type": {
      "_routing": {
        "required": true 
      }
    }
  }
}

PUT my_index/my_type/1 
{
  "text": "No routing value provided"
}
```

- (1) my_type文档需要路由。
- (2) <b>此索引请求会引发一个routing_missing_exception。</b>

##### 具有自定义路由的唯一ID

<b>
在索引指定自定义_routing的文档时，索引中的所有分片不能保证_id的唯一性。 事实上，如果使用不同的_routing值进行索引，那么具有相同_id的文档可能会在不同的分片上出现。

由用户确定ID在索引中是唯一的。
</b>

参考文章：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.4/mapping-routing-field.html
