---
title: "Flume学习（八）Flume解析自定义日志"
date: 2016-09-03 16:11:01
tags: [Flume]
categories: [Log]
---

### 环境简介

- JDK1.7.0_79
- Flume1.6.0
- Elasticsearch2.0.0

这里是基于上一篇《Flume学习（七）Flume整合Elasticsearch2.x》解析自定义的日志格式

### 解析的日志格式

这里由于篇幅原因，我简单列举了两条典型的日志格式

#### 日志文件格式

```
[{"name":"rp.api.call","request":"GET /api/test/settings","status":"succeeded","uid":889,"did":13,"duid":"app001","ua":"Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)","device_id":"65768768252343","ip":"10.190.1.67","server_timestamp":1463713488740}]
[{"name":"rp.api.call","request":"GET /api/test/search","errorStatus":200,"errorCode":"0000","errorMsg":"操作成功","status":"failed","uid":889,"did":13,"duid":"app002","ua":"Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)","device_id":"4543657687878989","ip":"10.190.1.66","server_timestamp":1463650301701}]
```

上一篇已经讲过了Flume解析日志格式主要使用interceptors，interceptors本身又支持多种type，这里我们主要介绍regex_extractor，即正则表达式匹配方式。下面的正则表达式可以匹配上面的两种格式的日志，上面两种日志格式的主要区别就是errorStatus，errorCode，errorMsg这三个字段有可能不存在，当没有报错的时候，这三个字段是不需要的。

#### 日志解析正则表达式

```
"name":(.*),"request":(.*),("errorStatus":(.*),)?("errorCode":(.*),)?("errorMsg":(.*),)?"status":(.*),"uid":(.*),"did":(.*),"duid":(.*),"ua":(.*),"device_id":(.*),"ip":(.*),"server_timestamp":([0-9]*)
```

但是在Flume的interceptors的regex表达式中配置上面的正则表达式会报错，我自己分析的原因是Flume的interceptor.serializers需要指定正则表达式拆分后的对应的字段和值，没有办法做到根据正则表达式动态处理。（这里我的分析可能不一定对，如果有疑问我们可以私下交流）

下面是我的解决办法，我根据上面两种日志格式写了两个interceptor，分别是es_interceptor和es_error_interceptor，每个interceptor对应不同的正则表达式，分别用来处理上面两种不同的日志格式。这样interceptor.serializers就能根据对应的正则表达式格式解析出来日志中对应的字段和值，再插入到ES索引中。


#### flume.conf配置文件

```
# 原始的正则表达式："name":(.*),"request":(.*),("errorStatus":(.*),)?("errorCode":(.*),)?("errorMsg":(.*),)?"status":(.*),"uid":(.*),"did":(.*),"duid":(.*),"ua":(.*),"device_id":(.*),"ip":(.*),"server_timestamp":([0-9]*)
agentX.sources.flume-avro-sink.interceptors = es_interceptor es_error_interceptor

agentX.sources.flume-avro-sink.interceptors.es_interceptor.type = regex_extractor
agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = "name":(.*),"request":(.*),"status":(.*),"uid":(.*),"did":(.*),"duid":(.*),"ua":(.*),"device_id":(.*),"ip":(.*),"server_timestamp":([0-9]*)

agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers = s1 s2 s3 s4 s5 s6 s7 s8 s9 s10
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s1.name = name
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s2.name = request
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s3.name = status
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s4.name = uid
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s5.name = did
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s6.name = duid
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s7.name = ua
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s8.name = device_id
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s9.name = ip
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s10.name = server_timestamp

agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.type = regex_extractor
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.regex = "name":(.*),"request":(.*),"errorStatus":(.*),"errorCode":(.*),"errorMsg":(.*),"status":(.*),"uid":(.*),"did":(.*),"duid":(.*),"ua":(.*),"device_id":(.*),"ip":(.*),"server_timestamp":([0-9]*)

agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers = s1 s2 s3 s4 s5 s6 s7 s8 s9 s10 s11 s12 s13
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s1.name = name
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s2.name = request
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s3.name = errorStatus
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s4.name = errorCode
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s5.name = errorMsg
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s6.name = status
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s7.name = uid
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s8.name = did
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s9.name = duid
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s10.name = ua
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s11.name = device_id
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s12.name = ip
agentX.sources.flume-avro-sink.interceptors.es_error_interceptor.serializers.s13.name = server_timestamp
```

#### ES的mapping如下

```
{
  "mappings": {
    "hb": {
      "properties": {
        "@fields": {
          "properties": {
            "uid": {
              "type": "string"
            },
            "duid": {
              "type": "string"
            },
            "status": {
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
            "did": {
              "type": "string"
            },
            "errorMsg": {
              "type": "string"
            },
            "device_id": {
              "type": "string"
            },
            "server_timestamp": {
              "type": "string"
            },
            "ip": {
              "type": "string"
            },
            "errorStatus": {
              "type": "string"
            }
          }
        },
        "@message": {
          "type": "string"
        }
      }
    }
  }
}
```

#### ES索引中的日志信息

```
{
	"_index": "test_log_index-2016-09-03",
	"_type": "test",
	"_id": "AVbvrWIPe8IcP1cQoXS2",
	"_version": 1,
	"_score": 1,
	"_source": {
		"@message": "[
			{"name":"1","request":"GET /api/test/settings","status":"succeeded","uid":889,"did":13,"duid":"app001","ua":"Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)","device_id":,"ip":"10.190.1.67","server_timestamp":1463713488740}
		]",
		"@fields": {
			"uid": "889",
			"duid": ""app001"",
			"status": ""succeeded"",
			"name": ""1"",
			"request": ""GET /api/test/settings"",
			"did": "13",
			"ua": ""Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)"",
			"device_id": "",
			"server_timestamp": "1463713488740",
			"ip": ""10.190.1.67""
		}
	}
}

{
	"_index": "test_log_index-2016-09-03",
	"_type": "test",
	"_id": "AVbvrWIPe8IcP1cQoXS3",
	"_version": 1,
	"_score": 1,
	"_source": {
		"@message": "[
			{"name":"rp.api.call","request":"GET /api/test/search","errorStatus":200,"errorCode":"0000","errorMsg":"操作成功","status":"failed","uid":889,"did":13,"duid":"app001","ua":"Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)","device_id":"4543657687878989","ip":"10.190.1.66","server_timestamp":1463650301701}
		]",
		"@fields": {
			"uid": "889",
			"status": ""failed"",
			"did": "13",
			"device_id": ""4543657687878989"",
			"errorMsg": ""操作成功"",
			"errorStatus": "200",
			"ip": ""10.190.1.66"",
			"duid": ""app001"",
			"request": ""GET /api/test/search"",
			"name": ""rp.api.call"",
			"errorCode": ""0000"",
			"ua": ""Dalvik/2.1.0 (Linux; U; Android 6.0.1; MI NOTE LTE MIUI/6.5.12)"",
			"server_timestamp": "1463650301701"
		}
	}
}
```

#### 总结

个人觉得这样的做法并不理想，因为日志格式肯定会多种多样，如果每种日志格式都需要不同的正则表达式来处理，显得太过笨重和冗余，因为刚接触Flume暂时没有发现有更好的做法，日后发现有更好的处理方式会重新更新上来。