---
title: "ELK学习（三）ELK处理URL参数"
date: 2017-01-24 11:16:23
tags: [ELK]
categories: [Log]
---

上一篇我们对Nginx的access.log进行了初步的解析和提取字段处理，如果想进一步对客户端的IP来源进行分析和地理定位，我们需要借助第三方库GeoIP来进行地理定位。

### 提取特殊字段

##### 提取URL参数

如果想要让URL参数也解析并且成为索引字段，比如一些通用参数，如uid, country, language, etc. 那么可以使用KV插件

```
filter {
    grok {
        ...
    }
    ...
    # 再单独将取得的URL、request字段取出来进行key-value值匹配
    # 需要kv插件。提供字段分隔符"&?"，值键分隔符"="，则会自动将字段和值采集出来。
    kv {
        source => "request" # 默认是message，我们这里只需要解析上面grok抽取出来的request字段
        field_split => "&?"
        value_split => "="
        include_keys => [ "network", "country", "language", "deviceId" ]
    }
　
    # 把所有字段进行urldecode（显示中文）
    urldecode {
        all_fields => true
    }
}
```

好了，现在还有一个问题，如果请求中有中文，那么日志中的中文是被urlencode之后存储的。我们具体分析的时候，比如有个接口是/api/search?keyword=我们，需要统计的是keyword被查询的热门顺序，那么就需要解码了。logstash牛逼的也有urldecode命令，urldecode可以设置对某个字段，也可以设置对所有字段进行解码。

```
urldecode {
    all_fields => true
}
```

##### 过滤掉安全扫描

对于安全扫描，只需要过滤 http_user_agent 中含有 inf-ssl-duty-scan 的请求就可以了

```
# 过滤安全扫描
if [http_user_agent] =~ "inf-ssl-duty-scan" {
    drop { }
}

# 将client_ip为"-"的转换成"0.0.0.0"，或者直接删除client_ip字段，两种方式的效果都一样，如果IP地址是"-"就会删除client_ip字段，否则geoip转换会报错
if [client_ip] == "-" {
    mutate {
        replace => { "client_ip" => "0.0.0.0" }
        # remove_field => [ "client_ip" ]
    }
}
```

### 查询Nginx请求日志

```
# 按照服务器响应时间区间查询请求
{"query":{"filtered":{"filter":{"bool":{"must":[{"range":{"upstream_response_time":{"gte":0.001,"lte":0.002}}}]}}}}}

# 按照整体响应时间区间查询请求
{"query":{"filtered":{"filter":{"bool":{"must":[{"range":{"request_time":{"gte":0.001,"lte":0.002}}}]}}}}}

# 查询某一个指定的请求
{"query":{"filtered":{"filter":{"bool":{"must":[{"term":{"fastcgi_script_name.raw":"/token/sign"}}]}}}}}

# 查询某一个指定的请求，服务器相应时间大于5s的
{"query":{"filtered":{"filter":{"bool":{"must":[{"range":{"upstream_response_time":{"gte":5}}},{"term":{"fastcgi_script_name.raw":"/token/sign"}}]}}}}}

# 查询某一IP端的请求
{"query":{"filtered":{"filter":{"bool":{"must":[{"range":{"client_ip":{"gte":"0.0.0.0"}}}]}}}}}

# 查询某一个IP的请求
{"query":{"filtered":{"filter":{"bool":{"must":[{"term":{"client_ip":"112.17.244.47"}}]}}}}}
```

参考文章：

- http://www.cnblogs.com/yjf512/p/4199105.html
- http://www.cnblogs.com/Orgliny/p/5592186.html
- http://blog.csdn.net/babauyang/article/details/8471090