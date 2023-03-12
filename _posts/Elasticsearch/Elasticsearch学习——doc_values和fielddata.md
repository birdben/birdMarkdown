---
title: "Elasticsearch学习——doc_values和fielddata"
date: 2017-07-16 15:43:42
tags: [Elasticsearch]
categories: [Search]
---

## doc_values（文档值）和fielddata（字段数据）

### 提纲

- doc_values

 * 什么是doc_values（文档值）
 * 为什么会使用doc_values
 * doc_values的原理
 * 什么时候产生doc_values
 * 什么时候使用doc_values

- fielddata

 * 什么是fielddata（字段数据）
 * 为什么会使用fielddata
 * fielddata的原理
 * 什么时候产生fielddata
 * 什么时候使用fielddata

- 总结

 * doc_values（文档值）和fielddata（字段数据）的比较
 * 我的发现

### 什么是doc_values（文档值）

简单说明doc_values就是一个种列式的数据存储结构，主要存储文档ID和词条集合的映射关系（doc_id -> terms），它存储于磁盘中。

### 为什么会使用doc_values

为什么会使用doc_values，先来说说倒排索引的不足之处。

doc_values的存在是因为倒排索引只对某些操作是高效的。 倒排索引的优势在于查找包含某个词条的文档，而对于从另外一个方向的相反操作并不高效，即：确定哪些词条是否存在单个文档里，聚合需要这种层级的访问模式。

倒排索引的优势在于查找包含某个项的文档，即通过term查找对应的doc_id。 所以倒排索引的检索性能是非常快的，但是在字段值排序时却不是理想的结构。因为当你对一个字段进行排序时，Elasticsearch需要访问每个匹配到的文档得到相关的值。

这也是为什么会引入doc_values的原因。

这里引用《Elasticsearch权威指南》中的例子（虽然我觉得这个例子并不恰当，后面会有具体的解释），我们有下倒排索引：

```
Term      Doc_1   Doc_2   Doc_3
------------------------------------
brown   |   X   |   X   |
dog     |   X   |       |   X
dogs    |       |   X   |   X
fox     |   X   |       |   X
foxes   |       |   X   |
in      |       |   X   |
jumped  |   X   |       |   X
lazy    |   X   |   X   |
leap    |       |   X   |
over    |   X   |   X   |   X
quick   |   X   |   X   |   X
summer  |       |   X   |
the     |   X   |       |   X
------------------------------------
```

如果我们想要获得所有包含``brown``的文档的词的完整列表，我们会创建如下查询：

```
GET /my_index/_search
{
  "query" : {
    "match" : {
      "body" : "brown"
    }
  },
  "aggs" : {
    "popular_terms": {
      "terms" : {
        "field" : "body"
      }
    }
  }
}
```

查询部分简单又高效。倒排索引是根据词条来排序的，所以我们首先在词条列表中找到``brown``，然后扫描所有的列（这里的列指的是上面的表格中的Doc_1，Doc_2，Doc_3），找到包含``brown``的文档。我们可以快速看到``Doc_1``和``Doc_2``包含``brown``这个token。

然后，对于聚合部分，我们需要找到``Doc_1``和``Doc_2``里所有唯一的词条。 用倒排索引做这件事情代价很高： 我们会迭代索引里的每个词条并收集``Doc_1``和``Doc_2``列里面token。这很慢而且难以扩展：随着词条和文档的数量增加，执行时间也会增加。

###### 注意

> 上面的表格实际上应该是作者方便读者理解才这么写的，实际上倒排索引的数据结构应该是词条和文档ID集合的映射关系。
> 
```
brown   |   (1,2)
dog     |   (1,3)
dogs    |   (2,3)
fox     |   (1,3)
foxes   |   (2)
in      |   (2)
jumped  |   (1,3)
lazy    |   (1,2)
leap    |   (2)
over    |   (1,2,3)
quick   |   (1,2,3)
summer  |   (2)
the     |   (1,3)
```
> 所以上面的查询的实际过程是
> 
> 1. 先在词条列表中找到``brown``，然后获得``brown``对应的文档ID集合（1和2），这样就先筛选出满足条件的文档ID了。
> 2. 对body字段聚合，我们需要知道上面满足查询条件的文档的所有body字段的值（也就是body字段都有哪些词条，因为body字段的值可能不止一个词条），所以需要遍历倒排索引中所有词条，检查他们对应的文档ID集合是否包含符合上面筛选条件的文档ID（1和2），如果包含就把该词条累加到聚合结果中。
> 
> 缺点已经很明显了，词条越多，文档数越多，查询时间就越长。这样也就证明倒排索引结构并不适合做字段的排序和聚合操作，也是为什么会有doc_values的原因。
> 
> 其实《Elasticsearch权威指南》中的这个doc_values的例子并不恰当，因为从上面的倒排索引结构可以看出实际上body字段是一个``analyzed ``的字段，所以每个文档中的body字段才会有这么多词条，这里本身是矛盾的，因为doc_values是不能用于``analyzed ``字段的。

### doc_values原理

doc_values通过转置（倒排索引中的词条和文档ID）两者间的关系来解决这个问题。倒排索引将词条映射到包含它们的文档ID集合，doc_values将文档ID映射到它们包含的词条集合：

```
Doc      Terms
-----------------------------------------------------------------
Doc_1 | brown, dog, fox, jumped, lazy, over, quick, the
Doc_2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
Doc_3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```

当数据被转置之后，想要收集到``Doc_1``和``Doc_2``的唯一token会非常容易。获得每个文档行，获取所有的词条，然后求两个集合的并集。

因此，搜索和聚合是相互紧密缠绕的。搜索使用倒排索引查找文档，聚合操作收集和聚合doc_values里的数据。

###### 注意

> 上面的表格实际上应该是作者方便读者理解才这么写的，实际上doc_values的数据结构应该是文档ID和词条集合的映射关系。
> 
```
-----------------------------------------------------------------
1 | brown, dog, fox, jumped, lazy, over, quick, the
2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```
> 所以上面的查询的实际过程变为
> 
> 1. 先在词条列表中找到``brown``，然后获得``brown``对应的文档ID集合（1和2），这样就先筛选出满足条件的文档ID了。
> 2. 对body字段聚合，我们需要知道上面满足查询条件的文档的所有body字段的值（也就是body字段都有哪些词条，因为body字段的值可能不止一个词条），因为有了doc_values结构，所以我们直接通过符合上面筛选条件的文档ID（1和2）就可以直接找到body字段包含的所有词条集合，然后对这些词条集合进行合并累加到聚合结果中。

### 什么时候产生doc_values

doc_values是在索引时与倒排索引同时产生的。也就是说doc_values是按段来产生的并且是不可变的，正如用于搜索的倒排索引一样。当字段索引时，Elasticsearch为了能够快速检索，会把字段的值加入倒排索引中，同时它也会存储该字段的doc_values。同样，和倒排索引一样，doc_values也序列化到磁盘。这些对于性能和伸缩性很重要。

通过序列化一个持久化的数据结构到磁盘，我们可以依赖于操作系统的缓存来管理内存，而不是在JVM堆栈里驻留数据。（这也是doc_values和fielddata的一个主要区别）

doc_values本质上是一个序列化的列式存储。

### 什么时候使用doc_values

doc_values不仅可以用于聚合。 任何需要查找某个文档包含的值的操作都必须使用它。 除了聚合，还包括排序，访问字段值的脚本，父子关系处理（参见 父-子关系文档 ）。

doc_values默认对所有字段启用，除了``analyzed ``字符类型字段。也就是说所有的数字、地理坐标、日期、IP 和不分析（ not_analyzed ）字符类型。

所以使用doc_values有个局限性，就是使用doc_values的字段不能是``analyzed ``。因为分析流程会产生很多新的token，这会让doc_values不能高效的工作。（ES 5.x版本中字段为text类型是不支持doc_values属性的）

如果你知道你永远也不会对某些字段进行聚合、排序或是使用脚本操作，可以对特定的字段禁用doc_values（尽管这种需求很罕见）。这会为你节省磁盘空间（因为doc_values再也没有序列化到磁盘），也许还能提升些许索引速度（因为不需要生成doc_values）。

通过设置``doc_values: false``，这个字段将不能被用于聚合、排序以及脚本操作。（其实这句话是不准确的，因为我们还可以使用fielddata，后面会详细介绍）

### 什么是fielddata（字段数据）

简单说明fielddata的数据结构与doc_values是相似的，也是存储的文档ID和词条集合的映射关系（doc_id -> terms），与doc_values不同之处是，它存储于内存中。因为fielddata能用于``analyzed ``字段，所以是倒排索引的``真正倒置``，又有人称作``正排索引``。

### 为什么会使用fielddata

其实是先有的fielddata，后来优化才出现的doc_values（ES是在1.4版本之后引入doc_values的，并且在2.x之后默认开启doc_values）。所以在doc_values没有出现之前，都是使用fielddata进行排序和聚合操作的。

### fielddata的原理

其实按照我的理解，上面doc_values的例子应该是fielddata的（我上面也曾提及到），因为doc_values本身是不支持``analyzed ``字段的，而fielddata是支持``analyzed ``字段的。所以fielddata在内存的实际数据结构如下：
 
```
-----------------------------------------------------------------
1 | brown, dog, fox, jumped, lazy, over, quick, the
2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```

与doc_values不同，fielddata构建和管理100%在内存中，常驻于JVM内存堆。这意味着它本质上是不可扩展的，有很多边缘情况下要提防。 

###### 注意

> 从历史上看，fielddata是所有字段的默认设置。但是Elasticsearch已迁移到doc_values以减少OOM 的几率。分析的字符串是仍然使用fielddata的最后一块阵地。最终目标是建立一个序列化的数据结构类似于doc_values，可以处理高维度的分析字符串，逐步淘汰fielddata。

### 什么时候产生fielddata

Elasticsearch加载内存fielddata的默认行为是延迟加载。如果你从来没有聚合一个分析字符串，就不会加载fielddata到内存中。此外，fielddata是基于字段加载的，这意味着只有很活跃地使用字段才会增加fielddata的负担。当Elasticsearch第一次查询某个字段时，它将会完整加载这个字段所有Segment中的倒排索引到内存中，以便于以后的查询能够获取更好的性能。

与doc_values不同，fielddata结构不会在索引时创建。相反，它是在查询运行时，动态填充。这可能是一个比较复杂的操作，可能需要一些时间。将所有的信息一次加载，再将其维持在内存中的方式要比反复只加载一个fielddata的部分代价要低。

加载fielddata有两种方式：

- 预加载 fielddata
- 延迟加载 fielddata（默认方式）

### 什么时候使用fielddata

答案很简单，当你要排序或者聚合的字段是``analyzed``的，就只能使用fielddata，因为doc_values不支持。当然这种情况比较少见，通常要排序和聚合的字段都是``not_analyzed``的，都可以使用doc_values来优化的。

还是使用《Elasticsearch权威指南》中的列子：

```
POST /agg_analysis/data/_bulk
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New Jersey" }
{ "index": {}}
{ "state" : "New Mexico" }
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New York" }
```

我们希望创建一个数据集里各个州的唯一列表，并且计数。简单，让我们使用terms桶：

```
GET /agg_analysis/data/_search
{
    "size" : 0,
    "aggs" : {
        "states" : {
            "terms" : {
                "field" : "state"
            }
        }
    }
}
```

得到结果：

```
{
...
   "aggregations": {
      "states": {
         "buckets": [
            {
               "key": "new",
               "doc_count": 5
            },
            {
               "key": "york",
               "doc_count": 3
            },
            {
               "key": "jersey",
               "doc_count": 1
            },
            {
               "key": "mexico",
               "doc_count": 1
            }
         ]
      }
   }
}
```

得到的聚合结果是state字段分析后的词条的数，而不是对各个州的名称的聚合数。

由此可以看出，聚合（fielddata和doc_values）是基于倒排索引创建的，倒排索引是 后置分析（ post-analysis ）的。

###### 注意

> 为什么聚合是基于倒排索引创建的，我的理解是聚合是基于fielddata和doc_values的，而fielddata和doc_values是基于倒排索引创建的。
> 
> - fielddata很好理解，当你对一个``analyzed``字段进行聚合操作，实际上就加载倒排索引并且生成fielddata的过程。当Elasticsearch第一次查询某个字段时，它将会完整加载这个字段所有Segment中的倒排索引到内存中，以便于以后的查询能够获取更好的性能。所以说fielddata是基于倒排索引创建的是没问题的，而且倒排索引是后置分析（ post-analysis ）的。
> - 对于doc_values可能理解起来有点怪，doc_values也是按段来产生的并且是不可变的，正如用于搜索的倒排索引一样。当字段索引时，Elasticsearch为了能够快速检索，会把字段（``not_analyzed``的）的值加入倒排索引中，同时它也会存储该字段的doc_values。准确来说，doc_values不能说是基于倒排索引创建的，只是doc_values的字段是不需要分析的，也就等同于后置分析（ post-analysis ）的（因为分析后还是整个词条），是一个特殊情况。

### doc_values（文档值）和fielddata（字段数据）的比较

-|doc_values|fielddata
---|---|---
存储结构|文档ID和词条集合的映射关系|文档ID和词条集合的映射关系
存储位置|磁盘|内存
字段支持|不支持``analyzed``字段|支持``analyzed``字段
产生时间|索引时产生|延迟加载/预加载
注意点|不能应用到text类型字段|需要注意内存使用和熔断

### 我的发现

我在《Elasticsearch权威指南》文档中发现一个问题，关于doc_values设置为false，《Elasticsearch权威指南》给出的说明如下：

```
By setting doc_values: false, this field will not be usable in aggregations, sorts or scripts
```

《Elasticsearch权威指南》原文链接：

- https://www.elastic.co/guide/en/elasticsearch/guide/current/_deep_dive_on_doc_values.html

《Elasticsearch权威指南》文档中的描述意思是如果某个字段的doc_values设置为false，那么该字段将不能进行排序，聚合和脚本执行。

但是在我的实际测试中发现即使在索引的mapping中指定某一个字段的doc_values设置为false，仍然可以对该字段进行排序和聚合。起初我也不明白是什么原因，也仔细翻阅了doc_values相关的文档，后来发现可能和fielddata有关，因为fielddata可以对analyzed的字段进行排序和聚合，对not_analyzed的字段应该也适用。猜测有可能在排序或者聚合的时候，动态的将所有的信息加载到fielddata。

```
从历史上看，fielddata 是 所有 字段的默认设置。但是 Elasticsearch 已迁移到 doc values 以减少 OOM 的几率。分析的字符串是仍然使用 fielddata 的最后一块阵地。 最终目标是建立一个序列化的数据结构类似于 doc values ，可以处理高维度的分析字符串，逐步淘汰 fielddata。
```

下面给出我测试的用例：

##### 在Elasticsearch 2.x版本中进行测试

- 创建一个doc_index的索引，包括code和name两个字段，name字段的doc_values设置为false。

```
$ curl -XPUT 'http://localhost:9200/doc_index' -d '{
  "mappings": {
    "my_type": {
      "properties": {
        "code": {
          "type":       "string",
          "index":      "not_analyzed"
        },
        "name": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false
        }
      }
    }
  }
}'
{"acknowledged":true}%
```

- 索引两个文档到doc_index索引中。

```
$ curl -XPUT 'http://localhost:9200/doc_index/my_type/1' -d '
{
  "code": "001",
  "name": "birdben"
}'
{"_index":"doc_index","_type":"my_type","_id":"1","_version":1,"_shards":{"total":2,"successful":1,"failed":0},"created":true}%


$ curl -XPUT 'http://localhost:9200/doc_index/my_type/2' -d '
{
  "code": "100",
  "name": "testuser"
}'
{"_index":"doc_index","_type":"my_type","_id":"2","_version":1,"_shards":{"total":2,"successful":1,"failed":0},"created":true}%
```

- 这里使用了按索引节点监控fielddata内存使用情况的API，发现doc_index索引的fielddata使用内存为0

```
$ curl -XGET 'http://localhost:9200/_nodes/stats/indices/fielddata?level=indices&fields=*&pretty'
{
  "cluster_name" : "ben-es",
  "nodes" : {
    "hNoFoYHwTaqhf1BgHSLlzQ" : {
      "timestamp" : 1494567623169,
      "name" : "Sleeper",
      "transport_address" : "172.17.0.2:9300",
      "host" : "172.17.0.2",
      "ip" : [ "172.17.0.2:9300", "NONE" ],
      "indices" : {
        "fielddata" : {
          "memory_size_in_bytes" : 0,
          "evictions" : 0,
          "fields" : { }
        },
        "indices" : {
          "doc_index" : {
            "fielddata" : {
              "memory_size_in_bytes" : 0,
              "evictions" : 0,
              "fields" : { }
            }
          }
        }
      }
    }
  }
}
```

- 我们先测试一下使用code进行排序，因为code字段的doc_values设置为true，所以doc_values是在索引数据时就已经创建好了，是不会通过fielddata来加载到内存中的。所以推测使用code排序后，doc_index索引的fielddata使用内存应该是没有变化的。

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "code": {
        "order": "asc"
      }
    }
  ]
}'
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [ {
      "_index" : "doc_index",
      "_type" : "my_type",
      "_id" : "1",
      "_score" : null,
      "_source" : {
        "code" : "001",
        "name" : "birdben"
      },
      "sort" : [ "001" ]
    }, {
      "_index" : "doc_index",
      "_type" : "my_type",
      "_id" : "2",
      "_score" : null,
      "_source" : {
        "code" : "100",
        "name" : "testuser"
      },
      "sort" : [ "100" ]
    } ]
  }
}
```

- 果然这里如我们前面推测的结果

```
$ curl -XGET 'http://localhost:9200/_nodes/stats/indices/fielddata?level=indices&fields=*&pretty'
{
  "cluster_name" : "ben-es",
  "nodes" : {
    "hNoFoYHwTaqhf1BgHSLlzQ" : {
      "timestamp" : 1494567654413,
      "name" : "Sleeper",
      "transport_address" : "172.17.0.2:9300",
      "host" : "172.17.0.2",
      "ip" : [ "172.17.0.2:9300", "NONE" ],
      "indices" : {
        "fielddata" : {
          "memory_size_in_bytes" : 0,
          "evictions" : 0,
          "fields" : { }
        },
        "indices" : {
          "doc_index" : {
            "fielddata" : {
              "memory_size_in_bytes" : 0,
              "evictions" : 0,
              "fields" : { }
            }
          }
        }
      }
    }
  }
}
```

- 接下来我们再试使用name字段进行排序，因为name字段的doc_values属性设置为false，所以默认的应该是使用fielddata延迟加载到JVM堆内存的

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "name": {
        "order": "asc"
      }
    }
  ]
}'
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [ {
      "_index" : "doc_index",
      "_type" : "my_type",
      "_id" : "1",
      "_score" : null,
      "_source" : {
        "code" : "001",
        "name" : "birdben"
      },
      "sort" : [ "birdben" ]
    }, {
      "_index" : "doc_index",
      "_type" : "my_type",
      "_id" : "2",
      "_score" : null,
      "_source" : {
        "code" : "100",
        "name" : "testuser"
      },
      "sort" : [ "testuser" ]
    } ]
  }
}
```

- 再次查看doc_index索引的fielddata使用内存情况，发现fielddata的内存使用已经从0变成了560

```
$ curl -XGET 'http://localhost:9200/_nodes/stats/indices/fielddata?level=indices&fields=*&pretty'
{
  "cluster_name" : "ben-es",
  "nodes" : {
    "hNoFoYHwTaqhf1BgHSLlzQ" : {
      "timestamp" : 1494567672018,
      "name" : "Sleeper",
      "transport_address" : "172.17.0.2:9300",
      "host" : "172.17.0.2",
      "ip" : [ "172.17.0.2:9300", "NONE" ],
      "indices" : {
        "fielddata" : {
          "memory_size_in_bytes" : 560,
          "evictions" : 0,
          "fields" : {
            "name" : {
              "memory_size_in_bytes" : 560
            }
          }
        },
        "indices" : {
          "doc_index" : {
            "fielddata" : {
              "memory_size_in_bytes" : 560,
              "evictions" : 0,
              "fields" : {
                "name" : {
                  "memory_size_in_bytes" : 560
                }
              }
            }
          }
        }
      }
    }
  }
}
```

说明我的推测没错，如果doc_values设置为false，fielddata默认是lazy，仍然会对name字段进行排序和聚合，只是不像doc_values一样在索引时就构建好，而是在查询的时候构建fielddata数据结构并且加载到JVM堆内存中，通过上面的fielddata查询接口也能看出在对name字段进行排序之后，fielddata的使用内存从0变成了560。

既然知道了原因，那么把fielddata设置为disabled后，应该就不可以对name字段进行排序了。

- 重新创建doc_index索引，这里将name字段的doc_values设置为false，并且将fielddata.format也设置为disabled

```
$ curl -XPUT 'http://localhost:9200/doc_index' -d '{
  "mappings": {
    "my_type": {
      "properties": {
        "code": {
          "type":       "string",
          "index":      "not_analyzed"
        },
        "name": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false,
          "fielddata": {
            "format": "disabled"
          }
        }
      }
    }
  }
}'
{"acknowledged":true}%
```

- 重新索引上面的两个文档到doc_index索引中。（具体命令省略了）

- 尝试对name字段进行排序

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "name": {
        "order": "asc"
      }
    }
  ]
}'
{
  "error" : {
    "root_cause" : [ {
      "type" : "illegal_state_exception",
      "reason" : "Field data loading is forbidden on [name]"
    } ],
    "type" : "search_phase_execution_exception",
    "reason" : "all shards failed",
    "phase" : "query",
    "grouped" : true,
    "failed_shards" : [ {
      "shard" : 0,
      "index" : "doc_index",
      "node" : "hNoFoYHwTaqhf1BgHSLlzQ",
      "reason" : {
        "type" : "illegal_state_exception",
        "reason" : "Field data loading is forbidden on [name]"
      }
    } ]
  },
  "status" : 500
}
```

发现报错：Field data loading is forbidden on [name]，说明我们的推断没错，doc_values设置为false，fielddata默认是lazy，仍然会对name字段进行排序和聚合。

##### 在Elasticsearch 5.x版本中进行测试
 
后来我又在ES 5.x版本中又进行了同样的测试，发现在使用同样的mapping映射，发现在创建mapping的时候ES 5.x已经自动将string类型转换成keyword和text类型了（因为ES 5.x版本已经不再支持string类型，由keyword和text替代）。

```
$ curl -XPUT 'http://localhost:9200/doc_index' -d '{
  "mappings": {
    "my_type": {
      "properties": {
        "code": {
          "type":       "string",
          "index":      "not_analyzed"
        },
        "name": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false
        }
      }
    }
  }
}'
```

- 查看mapping发现string类型已经被转换成keyword类型

```
$ curl -XGET 'http://localhost:9200/doc_index/_mapping?pretty'
{
  "doc_index" : {
    "mappings" : {
      "my_type" : {
        "properties" : {
          "code" : {
            "type" : "keyword"
          },
          "name" : {
            "type" : "keyword",
            "doc_values" : false
          }
        }
      }
    }
  }
}
```

- 此时我们再试使用name字段进行排序，因为name字段的doc_values属性设置为false，就会报错如下：

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "name": {
        "order": "asc"
      }
    }
  ]
}'
{
  "error" : {
    "root_cause" : [
      {
        "type" : "illegal_argument_exception",
        "reason" : "Can't load fielddata on [name] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
      }
    ],
    "type" : "search_phase_execution_exception",
    "reason" : "all shards failed",
    "phase" : "query",
    "grouped" : true,
    "failed_shards" : [
      {
        "shard" : 0,
        "index" : "doc_index",
        "node" : "dLWsPc-dSba1II7HrsAR8Q",
        "reason" : {
          "type" : "illegal_argument_exception",
          "reason" : "Can't load fielddata on [name] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
        }
      }
    ],
    "caused_by" : {
      "type" : "illegal_argument_exception",
      "reason" : "Can't load fielddata on [name] because fielddata is unsupported on fields of type [keyword]. Use doc values instead."
    }
  },
  "status" : 400
}
```

因为对keyword没有开启doc_values设置，不能使用fielddata加载name字段，因为fielddata不支持keyword类型。

- 我们将mapping稍作修改，将name的类型改成analyzed。

```
$ curl -XPUT 'http://localhost:9200/doc_index' -d '{
  "mappings": {
    "my_type": {
      "properties": {
        "code": {
          "type":       "string",
          "index":      "not_analyzed"
        },
        "name": {
          "type":       "string",
          "index":      "analyzed",
          "doc_values": false
        }
      }
    }
  }
}'
```

- 查看mapping发现string类型已经被转换成text类型

```
$ curl -XGET 'http://localhost:9200/doc_index/_mapping?pretty'
{
  "doc_index" : {
    "mappings" : {
      "my_type" : {
        "properties" : {
          "code" : {
            "type" : "keyword"
          },
          "name" : {
            "type" : "text"
          }
        }
      }
    }
  }
}
```

- 此时我们再试使用name字段进行排序，此时name字段现在text类型，会报错如下：

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "name": {
        "order": "asc"
      }
    }
  ]
}'
{
  "error" : {
    "root_cause" : [
      {
        "type" : "illegal_argument_exception",
        "reason" : "Fielddata is disabled on text fields by default. Set fielddata=true on [name] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
      }
    ],
    "type" : "search_phase_execution_exception",
    "reason" : "all shards failed",
    "phase" : "query",
    "grouped" : true,
    "failed_shards" : [
      {
        "shard" : 0,
        "index" : "doc_index",
        "node" : "dLWsPc-dSba1II7HrsAR8Q",
        "reason" : {
          "type" : "illegal_argument_exception",
          "reason" : "Fielddata is disabled on text fields by default. Set fielddata=true on [name] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
        }
      }
    ],
    "caused_by" : {
      "type" : "illegal_argument_exception",
      "reason" : "Fielddata is disabled on text fields by default. Set fielddata=true on [name] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory."
    }
  },
  "status" : 400
}
```

从报错信息中可以看出text类型的字段默认是不开启fielddata的，需要mapping中明确指定fielddata为true

- 我们再次修改mapping，将name字段的fielddata设置为true

```
$ curl -XPUT 'http://localhost:9200/doc_index' -d '{
  "mappings": {
    "my_type": {
      "properties": {
        "code": {
          "type":       "string",
          "index":      "not_analyzed"
        },
        "name": {
          "type":       "string",
          "index":      "analyzed",
          "fielddata": true
        }
      }
    }
  }
}'
```

- 先查看一下fielddata使用内存情况，fielddata没有使用内存

```
$ curl -XGET 'http://localhost:9200/_nodes/stats/indices/fielddata?level=indices&fields=*&pretty'
{
  "_nodes" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "cluster_name" : "ben-es",
  "nodes" : {
    "dLWsPc-dSba1II7HrsAR8Q" : {
      "timestamp" : 1500192369403,
      "name" : "node-1",
      "transport_address" : "172.17.0.2:9300",
      "host" : "172.17.0.2",
      "ip" : "172.17.0.2:9300",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "indices" : {
        "fielddata" : {
          "memory_size_in_bytes" : 0,
          "evictions" : 0,
          "fields" : {
            "_parent" : {
              "memory_size_in_bytes" : 0
            }
          }
        },
        "indices" : {
          "doc_index" : {
            "fielddata" : {
              "memory_size_in_bytes" : 0,
              "evictions" : 0,
              "fields" : { }
            }
          }
        }
      }
    }
  }
}
```

- 重复上面的步骤再次重试使用name字段进行排序，此时查询结果正常，说明使用了fielddata

```
$ curl -XPOST 'http://localhost:9200/doc_index/_search?pretty' -d '
{
  "query":{
     "match_all":{}
  },
  "sort":[
    {
      "name": {
        "order": "asc"
      }
    }
  ]
}'
{
  "took" : 73,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "doc_index",
        "_type" : "my_type",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "code" : "001",
          "name" : "birdben"
        },
        "sort" : [
          "birdben"
        ]
      },
      {
        "_index" : "doc_index",
        "_type" : "my_type",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "code" : "100",
          "name" : "testuser"
        },
        "sort" : [
          "testuser"
        ]
      }
    ]
  }
}
```

- 再查看一下fielddata使用内存情况，fielddata已经使用内存了

```
$ curl -XGET 'http://localhost:9200/_nodes/stats/indices/fielddata?level=indices&fields=*&pretty'
{
  "_nodes" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "cluster_name" : "ben-es",
  "nodes" : {
    "dLWsPc-dSba1II7HrsAR8Q" : {
      "timestamp" : 1500192369403,
      "name" : "node-1",
      "transport_address" : "172.17.0.2:9300",
      "host" : "172.17.0.2",
      "ip" : "172.17.0.2:9300",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "indices" : {
        "fielddata" : {
          "memory_size_in_bytes" : 560,
          "evictions" : 0,
          "fields" : {
            "name" : {
              "memory_size_in_bytes" : 560
            },
            "_parent" : {
              "memory_size_in_bytes" : 0
            }
          }
        },
        "indices" : {
          "doc_index" : {
            "fielddata" : {
              "memory_size_in_bytes" : 560,
              "evictions" : 0,
              "fields" : {
                "name" : {
                  "memory_size_in_bytes" : 560
                }
              }
            }
          }
        }
      }
    }
  }
}
```

因为开启了fielddata，所以，当name字段首次用于排序，聚合时，ES从倒排索引中查询出该字段的所有terms，并一致保存到内存中。

#### 总结

通过比较ES 2.x和ES 5.x的测试结果，ES 5.x已经对fielddata和doc_values的使用进行了强制限制，对keyword类型字段进行排序和聚合时，只能使用doc_values（默认开启，可以直接使用）。而对text类型字段进行排序和聚合时，只能使用fielddata（默认不开启，所以直接使用会报错）。这一点和ES 2.x有本质的区别。

参考文章：

- https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues-and-fielddata.html
- https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues-intro.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/doc-values.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/fielddata.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/keyword.html
- https://www.elastic.co/guide/en/elasticsearch/reference/5.4/text.html
- https://github.com/elastic/elasticsearch/issues/12394
- http://blog.h5min.cn/thomas0yang/article/details/64905926
- https://yq.aliyun.com/articles/6902
- http://www.linkedkeeper.com/detail/blog.action?bid=102
