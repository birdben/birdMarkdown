---
title: "Logstash学习（一）Logstash的Grok表达式"
date: 2016-11-23 12:52:53
tags: [Logstash]
categories: [Log]
---

今天遇到个比较奇葩的日志解析问题，我们的日志文件内容是标准的JSON格式的，但是在使用Logstash解析的时候，我们希望也能够保存原始的日志字符串到ES

### track.log日志文件

```
{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}
{"logs":[{"timestamp":"1475114827206","rpid":"65351516503932930","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840425}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.425Z"}
{"logs":[{"timestamp":"1475915077351","rpid":"65351516503932934","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090579}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.579Z"}
{"logs":[{"timestamp":"1475914816133","rpid":"65351516503932928","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829332}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.332Z"}
{"logs":[{"timestamp":"1475914827284","rpid":"65351516503932936","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840498}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.499Z"}
{"logs":[{"timestamp":"1475915077585","rpid":"65351516503932932","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090789}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.789Z"}
{"logs":[{"timestamp":"1475912701768","rpid":"65351516503932930","name":"birdben.api.call","request":"GET /api/test/settings","status":"succeeded","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}
{"logs":[{"timestamp":"1475913832349","rpid":"65351516503932934","name":"birdben.api.call","request":"GET /api/test/settings","errorStatus":200,"errorCode":"0000","errorMsg":"操作成功","status":"failed","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475913845544}],"level":"info","message":"logs","timestamp":"2016-10-08T08:04:05.544Z"}
{"logs":[{"timestamp":"1475915080561","rpid":"65351516503932928","name":"birdben.api.call","request":"GET /api/test/settings","errorStatus":200,"errorCode":"0000","errorMsg":"操作成功","status":"failed","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915093792}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:53.792Z"}
```

我开始的想法是从track.log日志文件中读取日志信息，因为track.log日志是标准的JSON格式的，所以直接将codec设置成json，因为logs是一个内嵌的数组，然后在filter根据logs做split，会把logs的数组拆分出多条日志信息，然后在匹配指定格式的timestamp并生成新的字段@timestamp。

```
input {    file {        path => ["/home/yunyu/Downloads/track.log"]        type => "api"
        codec => "json"        start_position => "beginning"        ignore_older => 0    }}filter {    split {        field => "logs"    }    date {        match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]        target => "@timestamp"    }}output {    stdout {        codec => rubydebug    }    elasticsearch {        codec => "json"        hosts => ["hadoop1:9200", "hadoop2:9200", "hadoop3:9200"]        index => "api_logs_index"        document_type => "%{type}"        workers => 1        flush_size => 20000        idle_flush_time => 10    }}
```

上述做法的确能够将track.log日志解析成功并且写入到ES中，但是没有办法获取到原日志信息，对我们来说是无法满足要求的。

为什么没办法获取到原日志信息呢？这里再次梳理下Logstash的处理流程。

原来一直以为是

- input --> filter --> output

应该补上decode/encode

- input --> decode --> filter --> encode --> output

decode/encode就是解码器和编码器

- input的codec是设置decode解码器
 * codec => json：把json字符串转换成json对象
- ouput的codec是设置encode编码器（大部分output都默认codec是json，例如：ES，Kafka等等）
 * codec => json：把json对象转换成json字符串

分析了一下codec => json的作用，就是直接输入预定义好的JSON数据，这样就可以省略掉filter/grok配置，其实就是可以省略在filter/grok中配置json插件配置了，codec默认值是plain，plain是一个空的解析器，它可以让用户自己指定格式。

```
input {
    file {
        ...
        codec => "json"
        ...
    }
}

filter {
}

等价于

input {
    file {
        ...
        codec => json {
            charset => ["UTF-8"] (optional), default: "UTF-8"
        }
        ...
    }
}

filter {
}

等价于

input {
    file {
        ...
        # 默认值是plain
        codec => "plain"
        ...
    }
}

filter {
    grok {
        match {
            "message" => "%{MATCH_ALL:@message}"
        }
    }
    json {
        source => "@message"
    }
}
```

MATCH_ALL是在{LOGSTASH_HOME}/patterns/postfix配置的grok表达式

```
# MATCH_ALL就是匹配任意的字符串
MATCH_ALL (.*)
```


也可以自己指定一个format格式，转换成String输出到指定端（ES，Kafka等等），注意format只对output生效

```
output {
    kafka {
        codec => plain {
            format => "%{message}"
        }
    }
}
```

但是，仍然不解为何使用codec => "json"后，filter就获取不到原日志信息

再次尝试，仍然使用codec => "json"，但是在filter中添加match表达式，通过match中的"message"来获取原日志信息，然后把值复制给@message新字段，后续进行split拆分等等其他操作。具体配置文件修改如下：

```
input {
    file {
        ...
        codec => "json"
        ...
    }
}

filter {    grok {        match => {            "message" => "%{MATCH_ALL:@message}"        }    }
    split {        field => "logs"    }    date {        match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]        target => "@timestamp"    }
}
```

尝试的结果并不是我们预期的，@message中获取的其实是我们track.log中的message字段的值"logs"。这里有个特殊情况就是我们track.log日志中带有message这个字段，但是为什么这里match表达式却获取到track.log日志中的message字段了呢？原因是我们input设置使用codec解码器为json（也就是将Logstash读取到我们file的原日志信息解析成json对象），match这里接收到的其实就是json对象中的message字段（就是我们track.log日志中的message字段），简单的理解match => {"message" => "%{MATCH_ALL:@message}"}就是通过"message"这个key获取filter接收的数据源（json对象或者原日志字符串）中的value，如果codec设置成json就是读取的json对象中的"message"的属性值，如果codec设置成plain，"message"是获取的原日志的字符串信息匹配grok表达式的值。

所以这里我们为了能够让match的"message"获取到原日志信息，而不是我们解析好的json日志中的message属性，我们把input的codec => "json"改成codec => "plain"，这样就会在input就将原日志解析成json对象了，而是我们在filter自己来处理。具体配置文件如下：

```
input {    file {        path => ["/home/yunyu/Downloads/track.log"]        type => "api"        start_position => "beginning"        ignore_older => 0    }}filter {    grok {        # MATCH_ALL为了匹配所有的字符串，然后将值复制给新字段@message        match => {            "message" => "%{MATCH_ALL:@message}"        }    }    # json的作用可以将日志字符串中是json字符串的部分解析转换成json对象    # 再将新字段@message转换成json对象    json {        source => "@message"    }    # split拆分转化好的json对象中的logs数组    split {        field => "logs"    }    date {        match => ["timestamp", "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"]        target => "@timestamp"    }}output {    stdout {        codec => rubydebug    }    elasticsearch {        codec => "json"        hosts => ["hadoop1:9200", "hadoop2:9200", "hadoop3:9200"]        index => "api_logs_index"        document_type => "%{type}"        workers => 1        flush_size => 20000        idle_flush_time => 10    }}
```

### ES中的索引数据

```
{
    "_index": "api_logs_index",
    "_type": "api",
    "_id": "AViQYVbFlnhRMZdfbXco",
    "_version": 1,
    "_score": 1,
    "_source": {
        "message": "logs",
        "@version": "1",
        "@timestamp": "2016-10-08T15:20:29.286Z",
        "path": "/home/yunyu/Downloads/birdben.log",
        "host": "hadoop1",
        "type": "api",
        "@message": "{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}",
        "logs": {
            "timestamp": "1475114816071",
            "rpid": "65351516503932932",
            "name": "birdben.ad.open_hb",
            "bid": 0,
            "uid": 0,
            "did": 0,
            "duid": 0,
            "hb_uid": 0,
            "ua": "",
            "device_id": "",
            "server_timestamp": 1475914829286
        },
        "level": "info",
        "timestamp": "2016-10-08T08:20:29.286Z"
    }
}
```

这里在使用split插件的过程中还发现了个Logstash的Bug，Split拆分的过程中如果遇到不是String和Array类型就会直接导致Logstash Crash，这里正确的做法应该类似grok一样，给对应的日志添加一个_jsonparsefailure的Tag，而不是导致Logstash直接崩溃，这个问题已经有人在GitHub上提了一个issue，貌似要等到Logstash 5.x的版本才会被修复。

具体地址：https://github.com/logstash-plugins/logstash-filter-split/issues/17


参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-filters-grok.html#plugins-filters-grok-match
- https://www.elastic.co/guide/en/logstash/2.3/plugins-codecs-json.html
- https://www.elastic.co/guide/en/logstash/2.3/plugins-filters-json.html#plugins-filters-json-source
- https://www.elastic.co/guide/en/logstash/2.3/plugins-filters-split.html#plugins-filters-split-field
- https://www.elastic.co/guide/en/logstash/2.3/configuration-file-structure.html#codec