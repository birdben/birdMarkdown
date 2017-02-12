---
title: "Logstash学习（一）elasticsearch插件——设置ES的Template"
date: 2016-12-22 14:55:58
tags: [Logstash]
categories: [Log]
---

我们使用ElasticSearch时一般需要自己创建ElasticSearch的索引的Mapping，当索引非常多的时候，可能需要配置一个索引模板Template来对类似的索引做统一配置，让索引模板Template中配置匹配索引的规则，来确定该Template会被应用到哪些索引上。

当Logstash在整合ElasticSearch的时候，会有下面三种方式的Template配置：

### Template配置方式

##### 1. 使用ElasticSearch默认自带的索引模板

ElasticSearch默认自带了一个名字为"logstash"的模板，默认应用于Logstash写入数据到ElasticSearch使用

- 优点：最简单，无须任何配置
- 缺点：无法自定义一些配置，例如：分词方式

##### 2. 在Logstash Indexer端自定义配置索引模板 

Logstash的output插件中使用template指定本机器上的一个模板json文件路径，可以在json文件中设置对应的Template模板信息。例如：template => "/tmp/logstash.json"

- 优点：配置简单
- 缺点：因为分散在Logstash Indexer机器上，维护起来比较麻烦

##### 3. 在ElasticSearch服务端自定义配置索引模板

由ElasticSearch负责加载模板。这种方式需要在ElasticSearch的集群中的config/templates路径下配置模板json。而且ElasticSearch提供了Restful API接口维护索引模板信息。

- 优点：维护比较容易，可动态更改，全局生效。
- 缺点：需要注意模板的命名规则，比较容易通过看Template名字就能够确定模板应用到哪些索引

这里可能使用第三种方式统一管理Template最好，推荐使用第三种方式，但是具体问题具体分析。例如我现在的场景就使用的第二种方式，因为我们的Logstash Indexer和ElasticSearch只有一台服务器，所以在Logstash Indexer端维护Template文件也可以。


### 模板类型

ElasticSearch的模板类型主要由两种：静态模板和动态模板

##### 静态模板

适合索引字段数据固定的场景，一旦配置完成，不能向里面加入多余的字段，否则会报错

- 优点：scheam已知，业务场景明确，不容易出现因字段随便映射从而造成元数据撑爆es内存，从而导致es集群全部宕机
- 缺点：字段数多的情况下配置稍繁琐，针对于每个索引可能需要的模板都不同，很有可能需要配置很多个模板

##### 动态模板

适合字段数不明确，大量字段的配置类型相同的场景，可以按照类型规则动态添加新字段，新加字段不会报错。主要需要配置"dynamic_templates"

- 优点：可动态添加任意字段，无须改动schema
- 缺点：无标准schema导致数据不规则，如果添加的字段非常多，有可能造成ES集群宕机

需要注意：模板在设置生效后，仅对ES集群中新建立的索引生效，而对已存在的索引及时索引名满足模板的匹配规则，也不会生效，因此如果需要改变现有索引的Mapping信息，仍需要在正确的Mapping基础上建立新的索引，并将数据从原索引拷贝至新索引，变更新索引别名为原索引这种方式来实现。

##### 模板结构

模板的结构大致分四部分：

第一部分：通用设置，主要是模板匹配索引的过滤规则，影响该模板对哪些索引生效；

第二部分：settings：配置索引的公共参数，比如索引的replicas，以及分片数shards等参数；

第三部分：mappings：最重要的一部分，在这部分中配置每个type下的每个field的相关属性，比如field类型（string,long,date等等），是否分词，是否在内存中缓存等等属性都在这部分配置；

第四部分：aliases：索引别名，索引别名可用在索引数据迁移等用途上。

```
{
  "logstash" : {
    "order" : 0,
    "template" : "logstash-*",
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "message_field" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string"
            },
            "match_mapping_type" : "string",
            "match" : "message"
          }
        }, {
          "string_fields" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string",
              "fields" : {
                "raw" : {
                  "ignore_above" : 256,
                  "index" : "not_analyzed",
                  "type" : "string"
                }
              }
            },
            "match_mapping_type" : "string",
            "match" : "*"
          }
        } ],
        "_all" : {
          "omit_norms" : true,
          "enabled" : true
        },
        "properties" : {
          "@timestamp" : {
            "type" : "date"
          },
          "geoip" : {
            "dynamic" : true,
            "properties" : {
              "ip" : {
                "type" : "ip"
              },
              "latitude" : {
                "type" : "float"
              },
              "location" : {
                "type" : "geo_point"
              },
              "longitude" : {
                "type" : "float"
              }
            }
          },
          "@version" : {
            "index" : "not_analyzed",
            "type" : "string"
          }
        }
      }
    },
    "aliases" : { }
  }
}
```

总结：

定制索引模板，是搜索业务中一项比较重要的步骤，需要注意的地方有很多，比如：

- 字段数固定吗
- 字段类型是什么
- 分不分词
- 索引不索引
- 存储不存储
- 排不排序
- 是否加权

除了这些还有其他的一些因素，比如，词库的维护改动，搜索架构的变化等等。如果前提没有充分的规划好，后期改变的话，改动其中任何一项，都需要重建索引，这个代价是非常大和耗时的，尤其是在一些数据量大的场景中。

### Logstash整合ElasticSearch模板实例

首先我们通过ElasticSearch的Restful API接口查询一下ElasticSearch中一共创建了多少个索引模板，默认情况下ElasticSearch应该会有一个名字为"logstash"的Template，这个Template匹配了所有"logstash-*"的索引，也就是说所有以"logstash-"开头的索引都默认使用了这个"logstash"的Template。这个Template就是一个动态模板。

```
curl -XGET localhost:9200/_template?pretty

{
  "logstash" : {
    "order" : 0,
    "template" : "logstash-*",
    "settings" : {
      "index" : {
        "refresh_interval" : "5s"
      }
    },
    "mappings" : {
      "_default_" : {
        "dynamic_templates" : [ {
          "message_field" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string"
            },
            "match_mapping_type" : "string",
            "match" : "message"
          }
        }, {
          "string_fields" : {
            "mapping" : {
              "fielddata" : {
                "format" : "disabled"
              },
              "index" : "analyzed",
              "omit_norms" : true,
              "type" : "string",
              "fields" : {
                "raw" : {
                  "ignore_above" : 256,
                  "index" : "not_analyzed",
                  "type" : "string"
                }
              }
            },
            "match_mapping_type" : "string",
            "match" : "*"
          }
        } ],
        "_all" : {
          "omit_norms" : true,
          "enabled" : true
        },
        "properties" : {
          "@timestamp" : {
            "type" : "date"
          },
          "geoip" : {
            "dynamic" : true,
            "properties" : {
              "ip" : {
                "type" : "ip"
              },
              "latitude" : {
                "type" : "float"
              },
              "location" : {
                "type" : "geo_point"
              },
              "longitude" : {
                "type" : "float"
              }
            }
          },
          "@version" : {
            "index" : "not_analyzed",
            "type" : "string"
          }
        }
      }
    },
    "aliases" : { }
  }
}
```

我们创建一个自定义Template动态模板，这个模板指定匹配所有以"go_logs_index_"开始的索引，并且指定允许添加新字段，匹配所有string类型的新字段会创建一个raw的嵌套字段，这个raw嵌套字段类型也是string，但是是not_analyzed不分词的（主要用于解决一些analyzed的string字段无法做统计，但可以使用这个raw嵌套字段做统计）

#### go_logs_template.json

```
{
  "template": "go_logs_index_*",
  "order":0,
  "settings": {
      "index.number_of_replicas": "1",
      "index.number_of_shards": "5",
      "index.refresh_interval" : "10s"
  },
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": false
      },
      "dynamic_templates": [
        {
          "my_template": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            }
          }
        }
      ]
    },
    "go": {
      "properties": {
        "timestamp": {
          "type": "string",
          "index": "not_analyzed"
        },
        "msg": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart"
        },
        "file": {
          "type": "string",
          "index": "not_analyzed"
        },
        "line": {
          "type": "string",
          "index": "not_analyzed"
        },
        "threadid": {
          "type": "string",
          "index": "not_analyzed"
        },
        "info": {
          "type": "string",
          "index": "not_analyzed"
        },
        "type": {
          "type": "string",
          "index": "not_analyzed"
        },
        "@timestamp": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "@version": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```

接下来我们看看Logstash是如何使用Template的。首先我们需要准备好Logstash要使用的Template文件，其实这里我们也可以在ES直接创建好Template，然后都在ES维护Template，就是前面说的推荐的第三种方式，但是对于我们单个Logstash Indexer和ElasticSearch节点，方式二和方式三配置都很简单。

在Logstash的配置文件中添加template相关的配置

```
...
output {
    stdout {
        codec => rubydebug
    }
    elasticsearch {
        codec => "json"
        hosts => ["hadoop1:9200", "hadoop2:9200", "hadoop3:9200"]
        index => "go_logs_index_%{+YYYY.MM.dd}"
        document_type => "%{type}"
        template => "/usr/local/elasticsearch/template/go_logs_template.json"
        template_name => "go_logs_template"
        template_overwrite => true
        workers => 1
        flush_size => 20000
        idle_flush_time => 10
    }
}

# 配置说明：
# template : 指定template模板文件
# template_name : 指定在ES中创建template的名称，默认是logstash
# template_overwrite : 是否覆盖ES中的template，默认是false
```

这样我们使用Logstash创建的索引"go_logs_index_%{+YYYY.MM.dd}"就会匹配到我们的Template，就会使用Template中的配置，所以使用Template之后就不需要每新创建一个索引就自己手动创建Mapping了，可以直接使用Template为一类的索引创建默认Mapping配置。


参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-outputs-elasticsearch.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/indices-templates.html#indices-templates
- http://www.cnblogs.com/JiaK/p/6134397.html
- http://qindongliang.iteye.com/blog/2290384
- http://xiaorui.cc/2015/12/17/elasticsearch如何修改mapping和template的方法/