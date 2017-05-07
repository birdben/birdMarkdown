---
title: "ELK学习（一）ELK收集Nginx日志"
date: 2017-01-23 17:35:14
tags: [ELK]
categories: [Log]
---

最近公司的系统遇到了并发性能问题，需要分析系统各个接口的请求和响应时间，所以提议收集Nginx日志来进行分析。


### Nginx日志格式

默认的Nginx配置文件/etc/nginx/nginx.conf

```
##
# Logging Settings
##
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" [$http_cookie] "$http_x_forwarded_for" $request_time $upstream_response_time $fastcgi_script_name';

access_log  /var/log/nginx/access.log  main;
error_log /var/log/nginx/error.log;
```

### 日志实例

下面是几个具有典型代表的access.log日志

```
10.XXX.XXX.XX - - [23/Jan/2017:18:54:19 +0800] "HEAD / HTTP/1.0" 204 0 "-" "-" [-] "-" 0.019 0.001 /
10.XXX.XXX.XX - - [23/Jan/2017:18:54:19 +0800] "HEAD / HTTP/1.0" 301 0 "-" "-" [-] "-" 0.018 - /
10.XXX.XXX.XX - - [23/Jan/2017:18:54:18 +0800] "GET /api/user/detail?ID=1234567890 HTTP/1.0" 200 1778 "-" "XYZ/2.1.1 (iPhone; iOS 10.2; Scale/2.00)" [-] "XXX.XXX.XX.XXX" 0.011 0.010 /api/user/detail
10.XXX.XXX.XX - - [23/Jan/2017:18:54:16 +0800] "POST /api/user/relation HTTP/1.0" 200 125 "-" "ABC/4.2.1 (iPhone; iOS 10.2; Scale/2.00)" [-] "XXX.XXX.XX.XXX" 0.073 0.071 /api/user/relation
```

看到上面的日志实例，可能已经看出来不同的日志差别还是挺大的，有些日志缺少client_ip, upstream_response_time, ua等等，这些日志是阿里云健康检查的日志，所以后面我们会想办法处理这些日志的。

### Nginx日志参数

```
$remote_addr : remote_addr代表客户端的IP，但它的值不是由客户端提供的，而是服务端根据客户端的IP指定的，当你的浏览器访问某个网站时，假设中间没有任何代理，那么网站的Web服务器（Nginx，Apache等）就会把remote_addr设为你的机器IP，如果你用了某个代理，那么你的浏览器会先访问这个代理，然后再由这个代理转发到网站，这样web服务器就会把remote_addr设为这台代理机器的IP。这里使用了阿里云的代理，所以是阿里云的内部IP地址。
$remote_user : 已经经过Auth Basic Module验证的用户名
$time_local : 通用日志格式下的本地时间
$request : 客户端请求的动作（通常为GET或POST），请求的URL（带参数），HTTP协议版本
$status : 请求状态
$body_bytes_sent : 发送给客户端的字节数，不包括响应头的大小； 该变量与Apache模块mod_log_config里的“%B”参数兼容。
$http_referer : 从哪个页面链接访问过来的
$http_user_agent : 记录客户端浏览器相关信息
$http_cookie : cookie的key和value
$http_x_forwarded_for : 正如上面所述，当你使用了代理时，Web服务器就不知道你的真实IP了，为了避免这个情况，代理服务器通常会增加一个叫做x_forwarded_for的头信息，把连接它的客户端IP（即你的上网机器IP）加到这个头信息里，这样就能保证网站的Web服务器能获取到真实IP。一般情况下有可能是多个IP地址，第一个就是客户端的真实IP地址。
$request_time : 指的就是从接受用户请求的第一个字节到发送完响应数据的时间，即包括接收请求数据时间、程序响应时间、输出响应数据时间。单位为秒，精度毫秒。
$upstream_response_time : 是指从Nginx向后端（php-cgi)建立连接开始到接受完数据然后关闭连接为止的时间。
$fastcgi_script_name : 请求的URL（不带参数）
```

注意：

从上面的描述可以看出，$request_time肯定比$upstream_response_time值大，特别是使用POST方式传递参数时，因为Nginx会把request body缓存住，接受完毕后才会把数据一起发给后端。所以如果用户网络较差，或者传递数据较大时，$request_time会比$upstream_response_time大很多。

### Grok表达式

Grok表达式的调试工具推荐使用

- http://grokdebug.herokuapp.com

这里定义了一个名为NGINX_ACCESS_LOGS的Grok表达式

```
NGUSERNAME [a-zA-Z\.\@\-\+_%]+
NGUSER %{NGUSERNAME}
NGINX_ACCESS_LOGS %{IPORHOST:server_ip} %{NGUSER:ident} %{NGUSER:auth} \[%{HTTPDATE:client_timestamp}\] "%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code} (?:%{NUMBER:http_bytes}|-) "(?:%{DATA:refer_url}|-)" "%{DATA:ua}" \[(?:%{GREEDYDATA:cookie}|-)\] "(?:%{IPORHOST:client_ip}|-)" (?:%{NUMBER:request_time}|-) (?:%{NUMBER:upstream_response_time}|-) %{GREEDYDATA:fastcgi_script_name}
```

Grok表达式的匹配结果如下：

```
{
  "NGINX_ACCESS_LOGS": [
    [
      "10.XXX.XXX.XX - - [23/Jan/2017:18:54:16 +0800] "POST /api/user/relation HTTP/1.0" 200 125 "-" "ABC/4.2.1 (iPhone; iOS 10.2; Scale/2.00)" [-] "XXX.XXX.XX.XXX" 0.073 0.071 /api/user/relation"
    ]
  ],
  "server_ip": [
    [
      "10.XXX.XXX.XX"
    ]
  ],
  "HOSTNAME": [
    [
      "10.XXX.XXX.XX",
      "XXX.XXX.XX.XXX"
    ]
  ],
  "IP": [
    [
      null,
      null
    ]
  ],
  "IPV6": [
    [
      null,
      null
    ]
  ],
  "IPV4": [
    [
      null,
      null
    ]
  ],
  "ident": [
    [
      "-"
    ]
  ],
  "NGUSERNAME": [
    [
      "-",
      "-"
    ]
  ],
  "auth": [
    [
      "-"
    ]
  ],
  "client_timestamp": [
    [
      "23/Jan/2017:18:54:16 +0800"
    ]
  ],
  "MONTHDAY": [
    [
      "23"
    ]
  ],
  "MONTH": [
    [
      "Jan"
    ]
  ],
  "YEAR": [
    [
      "2017"
    ]
  ],
  "TIME": [
    [
      "18:54:16"
    ]
  ],
  "HOUR": [
    [
      "18"
    ]
  ],
  "MINUTE": [
    [
      "54"
    ]
  ],
  "SECOND": [
    [
      "16"
    ]
  ],
  "INT": [
    [
      "+0800"
    ]
  ],
  "http_method": [
    [
      "POST"
    ]
  ],
  "url": [
    [
      "/api/user/relation"
    ]
  ],
  "http_version": [
    [
      "1.0"
    ]
  ],
  "BASE10NUM": [
    [
      "1.0",
      "200",
      "125",
      "0.073",
      "0.071"
    ]
  ],
  "status_code": [
    [
      "200"
    ]
  ],
  "http_bytes": [
    [
      "125"
    ]
  ],
  "refer_url": [
    [
      "-"
    ]
  ],
  "ua": [
    [
      "ABC/4.2.1 (iPhone; iOS 10.2; Scale/2.00)"
    ]
  ],
  "cookie": [
    [
      "-"
    ]
  ],
  "client_ip": [
    [
      "XXX.XXX.XX.XXX"
    ]
  ],
  "request_time": [
    [
      "0.073"
    ]
  ],
  "upstream_response_time": [
    [
      "0.071"
    ]
  ],
  "fastcgi_script_name": [
    [
      "/api/user/relation"
    ]
  ]
}
```

这里使用了Logstash默认定义的一些Grok表达式，下面列举了一些常用的Grok表达式

```
- IPORHOST : 匹配IP(IPv4或IPv6地址)
- HTTPDATE : 匹配时间(默认格式：01/Jan/2017:00:00:01 +0800)
- GREEDYDATA : 匹配字符串（.*）
- DATA : 匹配字符串（.*?）
- NUMBER : 匹配数字
- QS : 带引号的字符串
- WORD : 字符串，包括数字和大小写字母
- NOTSPACE : 不带任何空格的字符串
- SPACE : 空格字符串
- URIPROTO : URI协议
  比如：http、ftp等
- URIHOST : URI主机
  比如：www.baidu.com、10.10.1.11:8080等
- URIPATH : URI路径
  比如：//www.baidu.com/test/、/aaa.html等
- URIPARAM : URI里的GET参数
  比如：?a=1&b=2&c=3
- URIPATHPARAM : URI路径+GET参数
  比如：//www.baidu.com/test/aaa.html?a=1&b=2&c=3
- URI : 完整的URI
  比如：http://www.baidu.com/test/aaa.html?a=1&b=2&c=3
```

具体内置的Grok表达式可以参考：

- http://grokdebug.herokuapp.com/patterns#
- http://blog.csdn.net/tkchen/article/details/51815998

这里有几个地方让我有些迷惑：

- DATA和GREEDYDATA有什么区别
- client_ip是"-"的情况怎么处理
- upstream_response_time是"-"的情况怎么处理

##### DATA和GREEDYDATA的区别

实际上是下面正则表达式的区别

- .* : 贪婪模式
- .*? : 勉强模式
- .*+ : 侵占模式

具体区别请参考：

- http://stackoverflow.com/questions/3075130/what-is-the-difference-between-and-regular-expressions
- http://www.cnblogs.com/kevin-yuan/archive/2012/09/02/2667428.html

##### 如何处理client_ip是"-"的情况

在Logstash中匹配client_ip为"-"的情况，然后将client_ip字段去掉，这样geoip解析的时候就不会报错了，后续会提供详细的配置文件

##### 如何处理upstream_response_time是"-"的情况

upstream_response_time是"-"的情况比较好解决，只要稍微修改下Grok表达式即可，修改后如果upstream_response_time是"-"，Grok表达式提取出来的upstream_response_time的值就是null

```
修改前：%{NUMBER:upstream_response_time}
修改后：(?:%{NUMBER:upstream_response_time}|-)
```

以阿里云的健康检查请求日志为例：

```
10.XXX.XXX.XX - - [23/Jan/2017:18:54:19 +0800] "HEAD / HTTP/1.0" 301 0 "-" "-" [-] "-" 0.018 - /
```

匹配结果如下:

```
{
  "NGINX_ACCESS_LOGS": [
    [
      "10.XXX.XXX.XX - - [23/Jan/2017:18:54:19 +0800] "HEAD / HTTP/1.0" 301 0 "-" "-" [-] "-" 0.018 - /"
    ]
  ],
  "server_ip": [
    [
      "10.XXX.XXX.XX"
    ]
  ],
  "HOSTNAME": [
    [
      "10.XXX.XXX.XX",
      null
    ]
  ],
  "IP": [
    [
      null,
      null
    ]
  ],
  "IPV6": [
    [
      null,
      null
    ]
  ],
  "IPV4": [
    [
      null,
      null
    ]
  ],
  "ident": [
    [
      "-"
    ]
  ],
  "NGUSERNAME": [
    [
      "-",
      "-"
    ]
  ],
  "auth": [
    [
      "-"
    ]
  ],
  "client_timestamp": [
    [
      "23/Jan/2017:18:54:19 +0800"
    ]
  ],
  "MONTHDAY": [
    [
      "23"
    ]
  ],
  "MONTH": [
    [
      "Jan"
    ]
  ],
  "YEAR": [
    [
      "2017"
    ]
  ],
  "TIME": [
    [
      "18:54:19"
    ]
  ],
  "HOUR": [
    [
      "18"
    ]
  ],
  "MINUTE": [
    [
      "54"
    ]
  ],
  "SECOND": [
    [
      "19"
    ]
  ],
  "INT": [
    [
      "+0800"
    ]
  ],
  "http_method": [
    [
      "HEAD"
    ]
  ],
  "url": [
    [
      "/"
    ]
  ],
  "http_version": [
    [
      "1.0"
    ]
  ],
  "BASE10NUM": [
    [
      "1.0",
      "301",
      "0",
      "0.018",
      null
    ]
  ],
  "status_code": [
    [
      "301"
    ]
  ],
  "http_bytes": [
    [
      "0"
    ]
  ],
  "refer_url": [
    [
      "-"
    ]
  ],
  "ua": [
    [
      "-"
    ]
  ],
  "cookie": [
    [
      "-"
    ]
  ],
  "client_ip": [
    [
      null
    ]
  ],
  "request_time": [
    [
      "0.018"
    ]
  ],
  "upstream_response_time": [
    [
      null
    ]
  ],
  "fastcgi_script_name": [
    [
      "/"
    ]
  ]
}
```

### Logstash配置文件

这里我们尽量把处理方式简化，在单一的服务器上安装好Nginx和ELK环境，对Nginx的access.log日志进行收集，解析，最终写入到ES中。

```
input {
    file {
        path => ["/home/yunyu/Downloads/access.log"]
        type => "nginx_access"
        codec => "plain"
        start_position => "beginning"
        ignore_older => 0
    }
}

filter {
    if [type] == "nginx_access" {
        grok {
            match => {
                "message" => "%{NGINX_ACCESS_LOGS}"
            }
        }
        if "_grokparsefailure" in [tags] {
            drop { }
        }
        date {
            match => ["client_timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
            target => "@timestamp"
        }
    }
}

output {
    stdout {
        codec => rubydebug
    }
    if [type] == "nginx_access" {
        elasticsearch {
            codec => "json"
            hosts => ["hadoop1:9200", "hadoop2:9200", "hadoop3:9200"]
            index => "nginx_access_logs_index_%{+YYYY.MM.dd}"
            document_type => "%{type}"
            template => "/usr/local/elasticsearch/template/nginx_access_logs_template.json"
            template_name => "nginx_access_logs_template"
            template_overwrite => true
            workers => 1
            flush_size => 20000
            idle_flush_time => 10
        }
    }
}
```

### ES的索引Template模板

上面Logstash写入日志数据到ES的时候，使用了nginx_access_logs_template.json模板，下面是ES模板的具体配置。这里我们只是把Logstash提取出来的值按照ES的Template定义好的类型写入到ES索引中。

```
{
  "template": "nginx_access_logs_index_*",
  "order":0,
  "settings": {
      "index.number_of_replicas": "1",
      "index.number_of_shards": "5",
      "index.refresh_interval": "10s"
  },
  "mappings": {
    "_default_": {
      "_all": {
        "enabled": false
      },
      "dynamic_templates": [
        {
          "my_string": {
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
    "nginx_access": {
      "properties": {
        "ident": {
          "type": "string",
          "index": "not_analyzed"
        },
        "auth": {
          "type": "string",
          "index": "not_analyzed"
        },
        "client_ip": {
          "type": "ip"
        },
        "client_timestamp": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_method": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_version": {
          "type": "string",
          "index": "not_analyzed"
        },
        "url": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "refer_url": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "status_code": {
          "type": "string",
          "index": "not_analyzed"
        },
        "http_bytes": {
          "type": "string",
          "index": "not_analyzed"
        },
        "ua": {
          "type": "string",
          "analyzer": "ik",
          "search_analyzer": "ik_smart",
          "fields": {
            "raw": {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        },
        "device_id": {
          "type": "string",
          "index": "not_analyzed"
        },
        "sdk_version": {
          "type": "string",
          "index": "not_analyzed"
        },
        "host": {
          "type": "string",
          "index": "not_analyzed"
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

参考文章：

- https://www.digitalocean.com/community/tutorials/how-to-map-user-location-with-geoip-and-elk-elasticsearch-logstash-and-kibana
- http://kibana.logstash.es/content/logstash/examples/nginx-access.html