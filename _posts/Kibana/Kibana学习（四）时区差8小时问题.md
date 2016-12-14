---
title: "Kibana学习（四）时区差8小时问题"
date: 2016-12-05 18:35:36
tags: [Kibana]
categories: [Log]
---

Kibana连接ES查询数据的时候，会有时差8小时的问题。先来描述一下问题的具体情况，我们先来看看Logstash默认写入到ES的索引数据。timestamp是我们App上报日志的时间戳字段，这个字段是客户端写入日志的时间。@timestamp是使用Logstash写入ES的时候默认自带的时间戳（即Logstash生成ES索引的时间戳）。这里我们的这条日志是2016-10-08由App记录的，由Logstash收集到ES服务器的时间是2016-12-05。但是我们发现@timestamp并不是我们系统的当前时间，而是比我们当前的系统时间小了8小时，这就是我们想要解决的8小时时差问题。

原因是Logstash中默认插入的@timestamp时间字段是按照UTC 00:00标准时区的时间转换成long型的时间戳保存在ES中，而我们系统的时间是中国时区UTC +08:00，由ES查询出来的long型的时间戳再按照UTC +08:00转换成时间就正好相差了8小时。因为@timestamp的long型字段在ES中是不可取回的，所以我们在ES的返回值中是看不到这个long型字段的，只能看到@timestamp根据我们的时区转换后的结果"2016-12-05T11:30:40.911Z"，这个结果正好和我们的系统时间相差了8小时。

```
{
    "_index": "api_logs_index_1",
    "_type": "command",
    "_id": "AVjOwAt_jL-eD07Ygku1",
    "_version": 1,
    "_score": 1,
    "_source": {
        "logs": {
            "timestamp": "1475912701768",
            "rpid": "65351516503932930",
            "name": "rp.hb.ad.click_ad",
            "bid": 0,
            "uid": 0,
            "did": 0,
            "duid": 0,
            "hb_uid": 0,
            "ua": "",
            "device_id": "",
            "server_timestamp": 1475912715001
        },
        "level": "info",
        "message": "logs",
        "timestamp": "2016-10-08T07:45:15.001Z",
        "@version": "1",
        "@timestamp": "2016-12-05T11:30:40.911Z",
        "path": "/home/yunyu/Downloads/track.log",
        "host": "hadoop1",
        "type": "command"
    }
}
```

上面简单的描述了一下问题。我们需要解决下面两个问题：

1. 如何让Kibana查询和统计使用的是App写入日志的时间，而不是Logstash写入ES的时间（因为Logstash写入ES时间上会有延迟）
2. 解决时差相差8小时问题

一般我们App上报日志都会带有一个timestamp时间戳字段，这个字段是客户端写入日志的时间。当Logstash收集App上报日志的时候，会将timestamp字段保存到ES中，Kibana通过该字段当做统计的字段进行各种按日期统计的查询。

这里Kibana要求所有ES的索引必须要有一个时间字段作为统计查询的日期字段使用，如果没有ES的索引没有时间或者日期字段，是无法在Kibana中创建索引的。所以默认Kibana也会给一个默认的时间字段@timestamp，这样当我们在Kibana创建api_logs_index索引的时候，就会出现有两个时间字段，一个是timestamp，一个是@timestamp。

![Kibana](http://img.blog.csdn.net/20161205190259403?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在Kibana创建索引时，设置索引使用@timestamp字段，但是需要在Logstash中配置@timestamp的值从timestamp取出来的。Logstash中可以指定@timestamp字段的值是从App上报日志的timestamp字段来的。

```
date {
    match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]
    target => "@timestamp"
}
```

下面用我们这条日志举例，下面的表格是系统时间和UTC时间戳根据北京时区的转换对照表。这里的系统时间转换程UTC时间戳是带有北京时区的。

|系统时间	|UTC时间戳	|ES返回（显示）的时间	|ES存储的时间（long）
|-----		|------	|-----					|-----
|2016/10/18 15:45:15.001Z	|1475912715001	|											|timestamp : 1475912715001
|2016/10/18 7:45:15.001Z	|1475883915001	|timestamp : 2016/10/18 7:45:15.001Z	|@timestamp : 1475883915001
|2016/10/17 23:45:15.001Z	|				|@timestamp : 2016/10/17 23:45:15.001Z	|											|

Logstash将timestamp的时间2016/10/18 7:45:15.001Z按照默认的标准时区UTC 00:00将timestamp转换成long类型1475912715001存储到ES，而对于@timestamp字段的值，是将timestamp的时间2016/10/18 7:45:15.001Z按照北京时区UTC +08:00将timestamp转换成long类型1475883915001并且赋值给@timestamp并且存储到ES。（为什么要加上北京时区UTC +08:00的原因不清楚，也尝试过不带有Z时区设置的时间格式，转换成的时间戳结果一样）

```
date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
}
```

ES存储的时间long型到ES返回(显示)时间是按照UTC 00:00标准时间转换的。下面是取回的结果。

- ES将timestamp的long类型1475912715001按照UTC 00:00标准时间转换回时间2016/10/18 7:45:15.001Z
- ES将@timestamp的long类型1475883915001按照UTC 00:00标准时间转换回时间2016-10-07T23:45:15.001Z

```
{
    "_index": "api_logs_index_1",
    "_type": "command",
    "_id": "AVjOwAt_jL-eD07Ygku1",
    "_version": 1,
    "_score": 1,
    "_source": {
        "logs": {
            "timestamp": "1475912701768",
            "rpid": "65351516503932930",
            "name": "rp.hb.ad.click_ad",
            "bid": 0,
            "uid": 0,
            "did": 0,
            "duid": 0,
            "hb_uid": 0,
            "ua": "",
            "device_id": "",
            "server_timestamp": 1475912715001
        },
        "level": "info",
        "message": "logs",
        "timestamp": "2016-10-08T07:45:15.001Z",
        "@version": "1",
        "@timestamp": "2016-10-07T23:45:15.001Z",
        "path": "/home/yunyu/Downloads/track.log",
        "host": "hadoop1",
        "type": "command"
    }
}
```

这里可以看到Kibana的查询Request，下面是我选择的时间区间条件

- 2016/10/8 7:30:0：按照北京时区UTC +08:00转换为1475883000000
- 2016/10/8 8:0:0：按照北京时区UTC +08:00转换为1475884800000

Kibana Request

```
{
  "size": 500,
  "sort": [
    {
      "@timestamp": {
        "order": "desc",
        "unmapped_type": "boolean"
      }
    }
  ],
  "query": {
    "filtered": {
      "query": {
        "query_string": {
          "analyze_wildcard": true,
          "query": "*"
        }
      },
      "filter": {
        "bool": {
          "must": [
            {
              "range": {
                "@timestamp": {
                  "gte": 1475883000000,
                  "lte": 1475884800000,
                  "format": "epoch_millis"
                }
              }
            }
          ],
          "must_not": []
        }
      }
    }
  },
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
  "aggs": {
    "2": {
      "date_histogram": {
        "field": "@timestamp",
        "interval": "30s",
        "time_zone": "Asia/Shanghai",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": 1475883000000,
          "max": 1475884800000
        }
      }
    }
  },
  "fields": [
    "*",
    "_source"
  ],
  "script_fields": {},
  "fielddata_fields": [
    "timestamp",
    "@timestamp"
  ]
}
```

上面的Request可以看出来我们Kibana Request是将查询时间区间的条件先按照Browser的时区（北京时区UTC +08:00）转换成long型时间戳，然后当成条件查询的@timestamp字段。

这里的fields就是ES存储的long类型数值。

- timestamp 1475912715001：按照标准时区UTC +00:00转换为2016/10/8 7:45:15
- @timestamp 1475883915001：按照标准时区UTC +00:00转换为2016/10/7 23:45:15

```
"fields": {
  "timestamp": [
    1475912715001
  ],
  "@timestamp": [
    1475883915001
  ]
}
```


Kibana Response

```
{
  "took": 7,
  "hits": {
    "hits": [
      {
        "_index": "api_logs_index_1",
        "_type": "command",
        "_id": "AVjPhBGGUk8QUkLwTRVu",
        "_score": null,
        "_source": {
          "logs": {
            "timestamp": "1475912701768",
            "rpid": "65351516503932930",
            "name": "rp.hb.ad.click_ad",
            "bid": 0,
            "uid": 0,
            "did": 0,
            "duid": 0,
            "hb_uid": 0,
            "ua": "",
            "device_id": "",
            "server_timestamp": 1475912715001
          },
          "level": "info",
          "message": "logs",
          "timestamp": "2016-10-08T07:45:15.001Z",
          "@version": "1",
          "@timestamp": "2016-10-07T23:45:15.001Z",
          "path": "/home/yunyu/Downloads/track.log",
          "host": "hadoop1",
          "type": "command"
        },
        "fields": {
          "timestamp": [
            1475912715001
          ],
          "@timestamp": [
            1475883915001
          ]
        },
        "sort": [
          1475883915001
        ]
      }
    ],
    "total": 1,
    "max_score": 0
  },
  "aggregations": {
    "2": {
      "buckets": [
        {
          "key_as_string": "2016-10-08T07:30:00.000+08:00",
          "key": 1475883000000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:30:30.000+08:00",
          "key": 1475883030000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:31:00.000+08:00",
          "key": 1475883060000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:31:30.000+08:00",
          "key": 1475883090000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:32:00.000+08:00",
          "key": 1475883120000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:32:30.000+08:00",
          "key": 1475883150000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:33:00.000+08:00",
          "key": 1475883180000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:33:30.000+08:00",
          "key": 1475883210000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:34:00.000+08:00",
          "key": 1475883240000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:34:30.000+08:00",
          "key": 1475883270000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:35:00.000+08:00",
          "key": 1475883300000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:35:30.000+08:00",
          "key": 1475883330000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:36:00.000+08:00",
          "key": 1475883360000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:36:30.000+08:00",
          "key": 1475883390000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:37:00.000+08:00",
          "key": 1475883420000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:37:30.000+08:00",
          "key": 1475883450000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:38:00.000+08:00",
          "key": 1475883480000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:38:30.000+08:00",
          "key": 1475883510000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:39:00.000+08:00",
          "key": 1475883540000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:39:30.000+08:00",
          "key": 1475883570000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:40:00.000+08:00",
          "key": 1475883600000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:40:30.000+08:00",
          "key": 1475883630000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:41:00.000+08:00",
          "key": 1475883660000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:41:30.000+08:00",
          "key": 1475883690000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:42:00.000+08:00",
          "key": 1475883720000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:42:30.000+08:00",
          "key": 1475883750000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:43:00.000+08:00",
          "key": 1475883780000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:43:30.000+08:00",
          "key": 1475883810000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:44:00.000+08:00",
          "key": 1475883840000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:44:30.000+08:00",
          "key": 1475883870000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:45:00.000+08:00",
          "key": 1475883900000,
          "doc_count": 1
        },
        {
          "key_as_string": "2016-10-08T07:45:30.000+08:00",
          "key": 1475883930000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:46:00.000+08:00",
          "key": 1475883960000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:46:30.000+08:00",
          "key": 1475883990000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:47:00.000+08:00",
          "key": 1475884020000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:47:30.000+08:00",
          "key": 1475884050000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:48:00.000+08:00",
          "key": 1475884080000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:48:30.000+08:00",
          "key": 1475884110000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:49:00.000+08:00",
          "key": 1475884140000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:49:30.000+08:00",
          "key": 1475884170000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:50:00.000+08:00",
          "key": 1475884200000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:50:30.000+08:00",
          "key": 1475884230000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:51:00.000+08:00",
          "key": 1475884260000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:51:30.000+08:00",
          "key": 1475884290000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:52:00.000+08:00",
          "key": 1475884320000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:52:30.000+08:00",
          "key": 1475884350000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:53:00.000+08:00",
          "key": 1475884380000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:53:30.000+08:00",
          "key": 1475884410000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:54:00.000+08:00",
          "key": 1475884440000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:54:30.000+08:00",
          "key": 1475884470000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:55:00.000+08:00",
          "key": 1475884500000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:55:30.000+08:00",
          "key": 1475884530000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:56:00.000+08:00",
          "key": 1475884560000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:56:30.000+08:00",
          "key": 1475884590000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:57:00.000+08:00",
          "key": 1475884620000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:57:30.000+08:00",
          "key": 1475884650000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:58:00.000+08:00",
          "key": 1475884680000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:58:30.000+08:00",
          "key": 1475884710000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:59:00.000+08:00",
          "key": 1475884740000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T07:59:30.000+08:00",
          "key": 1475884770000,
          "doc_count": 0
        },
        {
          "key_as_string": "2016-10-08T08:00:00.000+08:00",
          "key": 1475884800000,
          "doc_count": 0
        }
      ]
    }
  }
}
```

ES返回的结果时间是按照标准时区UTC +00:00转换的。这里Kibana还会根据创建索引所选择的时间戳，再将@timestamp的结果转换成Browser默认的时区（即：UTC +08:00）显示出来，也就是在ES返回的@timestamp时间上再加8小时。

如果timestamp字段没有带有时区设置，我们可以在Logstash中指定locale和timezone属性来设置timestamp的字段的时区，这样就知道timestamp的时间应该按照指定的timezone时区解析成long型时间戳，这样my_timestamp字段在ES中是一个使用+08:00中国时区的long类型时间戳，方便使用ES API直接查询timestamp字段，因为一般人都使用+08:00中国时区将时间转换为long类型的时间戳。

```
date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
    target => "my_timestamp"
    locale => "en"
    timezone => "+08:00"
}

date {
    # 如果日志中的timestamp时间戳带有时区，而且是UTC标准时区，不是中国时区，我们可以使用指定timezone来转换成long类型的@timestamp
    match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]
    target => "@timestamp"
    locale => "en"
    timezone => "+00:00"
}
```

Logstash创建索引所使用的YYYY-MM-DD也是使用的@timestamp字段的时间


参考文章：

- https://discuss.elastic.co/t/elk-logstash-default-timezone-cause-index-splitting-problem-in-different-timezones/26615
- http://blog.csdn.net/bright_fu/article/details/48439367
- https://1024tools.com/timestamp