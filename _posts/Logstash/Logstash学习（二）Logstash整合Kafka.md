---
title: "Logstash学习（二）Logstash整合Kafka"
date: 2016-11-21 19:52:20
tags: [Logstash, Kafka]
categories: [Log]
---

前面我们已经实现通过Logstash读取track.log日志文件，然后写入到ES中。现在我们为了完善我们的日志收集系统架构，需要在中间添加Kafka消息队列做缓冲。这里我们使用了Logstash的Kafka插件来集成Kafka的。具体插件的官方地址如下：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-inputs-kafka.html
- https://www.elastic.co/guide/en/logstash/2.3/plugins-outputs-kafka.html

新版本的Logstash已经默认安装好大部分的插件了，所以无需像1.x版本的Logstash还需要手动修改Gemfile的source，然后手动安装插件了。

### 环境说明

- kafka_2.11-0.9.0.0
- zookeeper-3.4.8
- logstash-2.3.4
- elasticsearch-2.3.5

具体环境的安装这里不做重点介绍。

这里我们自己配置了一个logstash-shipper用来从track.log日志文件读取日志，并且写入到Kafka中。当然这里也可以由其他生产者来代替Logstash收集日志并且写入Kafka（比如：Flume等等）。这里我们是本地测试所以简单点直接使用Logstash读取本机的日志文件，然后写入到Kafka消息队列中。

### logstash-shipper-kafka.conf配置

```
input {
    file {
        path => ["/home/yunyu/Downloads/track.log"]
        type => "api"
        codec => "json"
        start_position => "beginning"
        # 设置是否忽略太旧的日志的
        # 如果没设置该属性可能会导致读取不到文件内容，因为我们的日志大部分是好几个月前的，所以这里设置为不忽略
        ignore_older => 0
    }
}

output {
    stdout {
        codec => rubydebug
    }
    kafka {
        # 指定Kafka集群地址
        bootstrap_servers => "hadoop1:9092,hadoop2:9092,hadoop3:9092"
        # 指定Kafka的Topic
        topic_id => "logstash_test"
    }
}
```

官网给出的注释

- ignore_older

The default behavior of the file input plugin is to ignore files whose last modification is greater than 86400s. To change this default behavior and process the tutorial file (which date can be much older than a day), we need to specify to not ignore old files.

### logstash-indexer-kafka.conf配置

```
input {
    kafka {
        # 指定Zookeeper集群地址
        zk_connect => "hadoop1:2181,hadoop2:2181,hadoop3:2181"
        # 指定当前消费者的group_id
        group_id => "logstash"
        # 指定消费的Topic
        topic_id => "logstash_test"
        # 指定消费的内容类型（默认是json）
        codec => "json"
        # 设置Consumer消费者从Kafka最开始的消息开始消费，必须结合"auto_offset_reset => smallest"一起使用
        reset_beginning => true
        # 设置如果Consumer消费者还没有创建offset或者offset非法，从最开始的消息开始消费还是从最新的消息开始消费
        auto_offset_reset => "smallest"
    }
}

filter {
    # 将logs数组对象进行拆分
    split {
        field => "logs"
    }
    date {
        match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]
        target => "@timestamp"
    }
}

output {
    stdout {
        codec => rubydebug
    }
    elasticsearch {
        codec => "json"
        hosts => ["hadoop1:9200", "hadoop2:9200", "hadoop3:9200"]
        index => "api_logs_index"
        workers => 1
        flush_size => 20000
        idle_flush_time => 10
    }
}
```

官网给出的注释

- auto\_offset\_reset

 * Value can be any of: largest, smallest
 * Default value is "largest"

 smallest or largest - (optional, default largest) If the consumer does not already have an established offset or offset is invalid, start with the earliest message present in the log (smallest) or after the last message in the log (largest).

- reset\_beginning

 * Value type is boolean
 * Default value is false

 Reset the consumer group to start at the earliest message present in the log by clearing any offsets for the group stored in Zookeeper. This is destructive! Must be used in conjunction with auto_offset_reset ⇒ smallest

### Mapping配置

```
{
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": true
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
    "api": {
      "properties": {
        "timestamp": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "message": {
          "type": "string",
          "index": "not_analyzed"
        },
        "level": {
          "type": "string"
        },
        "host": {
          "type": "string"
        },
        "logs": {
          "properties": {
            "uid": {
              "type": "long"
            },
            "status": {
              "type": "string"
            },
            "did": {
              "type": "long"
            },
            "device-id": {
              "type": "string"
            },
            "device_id": {
              "type": "string"
            },
            "errorMsg": {
              "type": "string"
            },
            "rpid": {
              "type": "string"
            },
            "url": {
              "type": "string"
            },
            "errorStatus": {
              "type": "long"
            },
            "ip": {
              "type": "string"
            },
            "timestamp": {
              "type": "string",
              "index": "not_analyzed"
            },
            "hb_uid": {
              "type": "long"
            },
            "duid": {
              "type": "string"
            },
            "request": {
              "type": "string"
            },
            "name": {
              "type": "string"
            },
            "errorCode": {
              "type": "string"
            },
            "ua": {
              "type": "string"
            },
            "server_timestamp": {
              "type": "long"
            },
            "bid": {
              "type": "long"
            }
          }
        },
        "path": {
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

Elasticsearch 会自动使用自己的默认分词器(空格，点，斜线等分割)来分析字段。分词器对于搜索和评分是非常重要的，但是大大降低了索引写入和聚合请求的性能。所以 logstash 模板定义了一种叫"多字段"(multi-field)类型的字段。这种类型会自动添加一个 ".raw" 结尾的字段，并给这个字段设置为不启用分词器。简单说，你想获取 url 字段的聚合结果的时候，不要直接用 "url" ，而是用 "url.raw" 作为字段名。

这里使用dynamic_templates是因为我们这里有嵌套结构logs，即使我们在内嵌的logs结构中定义了字段是not_analyzed，但是新创建出来的索引数据仍然是analyzed的（不知道是为什么）。如果字段都是analyzed就无法在Kibana中进行统计，这里使用dynamic_templates，给所有动态字段都加一个raw字段，这个字段名就是原字段（比如:logs.name）后面加上一个.raw（变成logs.name.raw），专门用来解决analyzed无法做统计的，所有的.raw字段都是not_analyzed，这样就可以使用.raw字段（logs.name.raw）进行统计分析了，而全文搜索可以继续使用原字段（logs.name）。

这里还需要注意的就是，需要精确匹配的字段要设置成not_analyzed（例如：某些ID字段，或者可枚举的字段等等），需要全文搜索的字段要设置成analyzed（例如：日志详情，或者具体错误信息等等），否则在Kibana全文搜索的时候搜索结果是正确的，但是没有高亮，就是因为全文搜索默认搜索的是_all字段，高亮结果返回却是在_source字段中。还有Kibana的全文搜索默认是搜索的_all字段，需要在ES创建mapping的时候设置_all开启状态。

Highlight高亮不能应用在非String类型的字段上，必须把integer，long等非String类型的字段转化成String类型来创建索引，这样这些字段才能够被高亮搜索。

还有就是记得每次修改完ES Mapping文件要刷新Kibana中的索引

最终修改后的ES Mapping如下：

```
{
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": true
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
    "api": {
      "properties": {
        "timestamp": {
          "format": "strict_date_optional_time||epoch_millis",
          "type": "date"
        },
        "message": {
          "type": "string",
          "index": "not_analyzed"
        },
        "level": {
          "type": "string",
          "index": "not_analyzed"
        },
        "host": {
          "type": "string",
          "index": "not_analyzed"
        },
        "logs": {
          "properties": {
            "uid": {
              "type": "string",
              "index": "not_analyzed"
            },
            "status": {
              "type": "string",
              "index": "not_analyzed"
            },
            "did": {
              "type": "long",
              "fields": {
                "as_string": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "device-id": {
              "type": "string",
              "index": "not_analyzed"
            },
            "device_id": {
              "type": "string",
              "index": "not_analyzed"
            },
            "errorMsg": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "rpid": {
              "type": "string",
              "index": "not_analyzed"
            },
            "url": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "errorStatus": {
              "type": "long",
              "fields": {
                "as_string": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "ip": {
              "type": "string",
              "index": "not_analyzed"
            },
            "timestamp": {
              "type": "string",
              "index": "not_analyzed"
            },
            "hb_uid": {
              "type": "long",
              "fields": {
                "as_string": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "duid": {
              "type": "string",
              "index": "not_analyzed"
            },
            "request": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "name": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "errorCode": {
              "type": "string",
              "index": "not_analyzed"
            },
            "ua": {
              "type": "string",
              "fields": {
                "raw": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            },
            "server_timestamp": {
              "type": "long"
            },
            "bid": {
              "type": "long",
              "fields": {
                "as_string": {
                  "type": "string",
                  "index": "not_analyzed"
                }
              }
            }
          }
        },
        "path": {
          "type": "string",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
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

name，request，path，ua，url，errorMsg虽然设置了analyzed，但是同时也需要做统计，所以在这几个字段单独加上了.raw字段，用来统计使用。

这里有个小技巧就是我们没有直接把long类型的字段直接转换成String类型，我们是在这个long类型的字段下创建了一个as_string字段，as_string这个字段是String类型的，并且是not_analyzed，这样Kibana在全文搜索的时候就会高亮出来long类型的字段了，实际上是高亮的long类型字段下的String字段。举例：下面是搜索一个logs.bid字段，logs.bid这个字段是long类型的，但是我们在这个字段下创建了一个logs.bid.as_string字段，实际上highlight高亮的字段也是logs.bid.as_string这个字段。

参考：

- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/multi-fields.html

```
...
"highlight": {
  "logs.duid": [
    "@kibana-highlighted-field@wasl6@/kibana-highlighted-field@"
  ],\
  "logs.bid.as_string": [
    "@kibana-highlighted-field@79789714801950720@/kibana-highlighted-field@"
  ],
  "type": [
    "@kibana-highlighted-field@api@/kibana-highlighted-field@"
  ],
  "logs.request": [
    "GET /@kibana-highlighted-field@api@/kibana-highlighted-field@/hongbao/realname/info"
  ]
}
...
```

### Kibana查询Request

```
{
  "size": 500,
  "highlight": {
    "pre_tags": [
      "@kibana-highlighted-field@"
    ],
    "post_tags": [
      "@/kibana-highlighted-field@"
    ],
    "fields": {
      "*": {}
    },
    "require_field_match": false,
    "fragment_size": 2147483647
  },
  "query": {
    "filtered": {
      "query": {
        "query_string": {
          "query": "keyword",
          "analyze_wildcard": true
        }
      }
    }
  },
  "fields": [
    "*",
    "_source"
  ]
}
```

这里Kibana全文搜索使用的是query_string语法，下面是常用的参数

- query：可以使用简单的Lucene语法
- default_field：指定默认查询哪些字段，默认值是_all
- analyze_wildcard：默认情况下，通配符查询是不会被分词的，如果该属性设置为true，将尽力去分词。（原文：By default, wildcards terms in a query string are not analyzed. By setting this value to true, a best effort will be made to analyze those as well.）

下面是ES官方文档的相关说明

```
Wildcards
Wildcard searches can be run on individual terms, using ? to replace a single character, and * to replace zero or more characters:

qu?ck bro*
Be aware that wildcard queries can use an enormous amount of memory and perform very badly — just think how many terms need to be queried to match the query string "a* b* c*".

Warning
Allowing a wildcard at the beginning of a word (eg "*ing") is particularly heavy, because all terms in the index need to be examined, just in case they match. Leading wildcards can be disabled by setting allow_leading_wildcard to false.

Wildcarded terms are not analyzed by default — they are lowercased (lowercase_expanded_terms defaults to true) but no further analysis is done, mainly because it is impossible to accurately analyze a word that is missing some of its letters. However, by setting analyze_wildcard to true, an attempt will be made to analyze wildcarded words before searching the term list for matching terms.
```

### 遇到的问题和解决方法

Q : 公司之前的架构是Flume + KafKa + Logstash + ES，但是使用Flume作为Shipper端添加相关的type、host、path等Header字段会按照StringSerializer序列化到Kafka中，但是Logstash无法解析Flume序列化后的Header字段
A : 将Shipper端换成Logstash，保证Shipper和Indexer用同样的序列化和反序列化方式。

Q : 最近部署了线上的logstash，发现一个问题ES的host字段为0.0.0.0，这个host是Logstash Shipper端自动添加的Header字段。
A : 后来发现是因为/etc/hosts的IP、主机名和hostname不一致导致的, 只要设置成一致就可以解决这个问题了。

参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-inputs-kafka.html
- https://www.elastic.co/guide/en/logstash/2.3/plugins-outputs-kafka.html
- http://udn.yyuap.com/doc/logstash-best-practice-cn/output/elasticsearch.html
- http://elasticsearch.cn/article/23
- http://stackoverflow.com/questions/36715849/numeric-and-date-fields-highlighting-in-elastic-search
- http://stackoverflow.com/questions/36715688/indexing-numeric-field-as-both-int-and-string-in-elastic-search
- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/query-dsl-query-string-query.html
- https://www.elastic.co/guide/en/elasticsearch/reference/2.3/multi-fields.html
- http://www.v1en.com/2016/03/31/logstash-host-field-is-0-0-0-0/