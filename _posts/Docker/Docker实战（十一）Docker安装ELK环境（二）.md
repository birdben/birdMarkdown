---
title: "Docker实战（十一）Docker安装ELK环境（二）"
date: 2016-01-08 06:40:33
tags: [Docker命令, Dockerfile, ELK]
categories: [Docker]
---

日志收集系统架构
![日志收集系统架构](http://img.blog.csdn.net/20160103152529612?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html
- http://dockone.io/article/505#rd?sukey=fc78a68049a14bb21970769bd4cdecc8f803567ce85b9148150a8f086c5ea01b195498f33b858ddf9c93183f1c09d255
- http://mp.weixin.qq.com/s?__biz=MzA4NDM0OTQ0NA==&mid=400670251&idx=1&sn=dec80ffddf9b6b0619a89605c397a2c2&scene=1&srcid=1201EdUMaGhzFetS4f7Pe8QX&key=ac89cba618d2d9767d9841c0e75232b77db8e3d08ed0c4bd45b9796dc6c27292d6f9776642a755bcf64dfeededbede63&ascene=0&uin=OTIxOTMxODgw&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.10.3+build(14D136)&version=11020201&pass_ticket=wAdeTXAnaEoTmxMJDYGfZiYPEQFQiUuMEieo5%2F72eVKwxLRWIshfT3nzlOwtjLDy

##### 日志收集系统架构简介

- Logstash Agent/Flume：采集各个业务系统的日志，然后发送给Redis/Kafka消息队列。
- Redis/Kafka：接收用户日志的消息队列，临时存储日志并且起到缓冲的作用，防止日志量上来之后，拖垮Logstash Indexer。
- Logstash Indexer：做日志解析，统一成JSON输出给Elasticsearch。
- Elasticsearch：实时日志分析服务的核心技术，一个schemaless，实时的数据存储服务，通过index组织数据，兼具强大的搜索和统计功能。
- Kibana：基于Elasticsearch的数据可视化组件，超强的数据可视化能力是众多公司选择ELK stack的重要原因。

这里实现的是 ELK + Redis 收集 Nginx 的 access.log
我们这里是把各个组件拆开，模拟每个组件是一个集群，每个集群部署在一台机器上

1.Nginx + Logstash agent
2.Redis
3.Logstash indexer
4.ElasticSearch
5.Kibana

##### Nginx配置文件

```
daemon off;
master_process off;
error_log  logs/error.log;

...

http {
    include       mime.types;
    default_type  application/octet-stream;
    
    # 设置nginx日志格式，格式的名称logstash
    log_format  logstash  '$http_host $server_addr $remote_addr [$time_local] "$request" $request_body $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_time $upstream_response_time';
    
    # 设置access_log日志输出的文件路径，以及使用的日志格式名称
    access_log  logs/access.log  logstash;
    
    # 以下内容省略
    ...
}
```

##### logstash agent的配置文件logstash.conf
```
input {
        file {
                type => "nginx_access"
                path => ["/usr/local/nginx/logs/access.log"]
        }
}
output {
        redis {
                host => "10.211.55.4"
                port => 6379
                password => admin
                data_type => "list"
                key => "logstash:redis"
        }
}
```

##### 注意
这里配置就是往redis队列中塞入日志就行，所以input的来源是Nginx的log文件，output的目标设置为redis，这里redis充当MQ消息队列的作用。

##### logstash indexer的配置文件logstash.conf
```
input {
		redis {
				host => "10.211.55.4"
				port => 6379
				password => admin
				data_type => "list"
				key => "logstash:redis"
		}
}

filter {
		grok {
				match => [
					"message", "%{IPORHOST:http_host} %{IPORHOST:server_ip} %{IPORHOST:client_ip} \[%{HTTPDATE:timestamp}\] \"%{WORD:http_verb} (?:%{PATH:baseurl}\?%{NOTSPACE:params}(?: HTTP/%{NUMBER:http_version})?|%{DATA:raw_http_request})\" (%{NOTSPACE:params})?|- %{NUMBER:http_status_code} (?:%{NUMBER:bytes_read}|-) %{QS:referrer} %{QS:agent} %{NUMBER:time_duration:float} %{NUMBER:time_backend_response:float}"
			]
		}
		kv {
				prefix => "params."
				field_split => "&"
				source => "params"
		}
		urldecode {
				all_fields => true
		}
		date {
				locale => "en"
				match => ["timestamp" , "dd/MMM/YYYY:HH:mm:ss Z"]
		}
}

output {
		elasticsearch {
				embedded => false				codec => "json"				protocol => "http"				host => "10.211.55.4"				port => 9200				index => "birdlogstash"		}
}
```

##### 注意
logstash indexer的配置文件就比较麻烦了，需要配置的有三个部分

- input: 负责从redis中获取日志数据
- filter: 负责对日志数据进行分析和结构化
- output: 负责将结构化的数据存储进入elasticsearch

这里配置就是从redis队列中获取日志，对日志进行相应地过滤，所以input的来源是Redis的list队列，其中的redis配置当然要和logstash agent的配置一致了。output的目标设置为ES搜索引擎的索引，在通过kibana图形界面化的展示ES的查询索引。


##### kibana.yml配置文件
```
# 这里配置Kibana访问ES集群的地址
# The Elasticsearch instance to use for all your queries.
elasticsearch_url: "http://10.211.55.4:9200"
```

##### elk_log_agent的Dockerfile文件
```
############################################
# version : birdben/elk_log_agent:v1
# desc : 当前版本安装的elk_log_agent
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 LOGSTASH 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV LOGSTASH_HOME /software/logstash-1.5.4

# 复制 logstash-1.5.4 文件到镜像中（logstash-1.5.4 文件夹要和Dockerfile文件在同一路径）
ADD logstash-1.5.4 /software/logstash-1.5.4

# 解决环境问题，否则logstash无法从log文件中采集日志。具体环境： Logstash 1.5, Ubuntu 14.04, Oracle JDK7
RUN ln -s /lib/x86_64-linux-gnu/libcrypt.so.1 /usr/lib/x86_64-linux-gnu/libcrypt.so

# 安装升级gcc
RUN sudo rm -rf /var/lib/apt/lists/*
RUN sudo apt-get update

RUN sudo apt-get -y install \
build-essential

RUN sudo mkdir -p /software/temp
RUN wget http://nginx.org/download/nginx-1.8.0.tar.gz && tar -zxvf nginx-1.8.0.tar.gz -C /software/temp
RUN wget http://zlib.net/zlib-1.2.8.tar.gz && tar -zxvf zlib-1.2.8.tar.gz -C /software/temp
RUN wget http://www.openssl.org/source/openssl-1.0.1q.tar.gz && tar -zxvf openssl-1.0.1q.tar.gz -C /software/temp
RUN wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz && tar -zxvf pcre-8.37.tar.gz -C /software/temp
RUN cd /software/temp/nginx-1.8.0 && sudo ./configure --sbin-path=/software/nginx-1.8.0/nginx --conf-path=/software/nginx-1.8.0/nginx.conf --pid-path=/software/nginx-1.8.0/nginx.pid --with-http_ssl_module --with-pcre=/software/temp/pcre-8.37 --with-zlib=/software/temp/zlib-1.2.8 --with-openssl=/software/temp/openssl-1.0.1q && sudo make && sudo make install

# 设置nginx是非daemon启动
RUN echo 'daemon off;' >> /software/nginx-1.8.0/nginx.conf
RUN echo 'master_process off;' >> /software/nginx-1.8.0/nginx.conf
RUN echo 'error_log  logs/error.log;' >> /software/nginx-1.8.0/nginx.conf

# 设置 NGINX 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV NGINX_HOME /software/nginx-1.8.0

# 挂载/logstash目录
VOLUME ["/logstash"]

# 容器需要开放Nginx 80端口
EXPOSE 80

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### elk_log_agent的supervisord.conf配置文件
```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:nginx]
command=/software/nginx-1.8.0/nginx -c /software/nginx-1.8.0/nginx.conf

[program:logstash]
# 指定配置文件时，一定要使用绝对路径，相对路径是不好用的，这个坑已经踩过两次了。。
command=/software/logstash-1.5.4/bin/logstash -f /logstash/logstash_agent.conf
```

##### elk_log_agent的控制台终端
```
# 构建镜像
$ docker build -t="birdben/elk_log_agent:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 8888:80 -t -i -v /docker/logstash:/logstash "birdben/elk_log_agent:v1"
```

##### logstash的Dockerfile文件
```
############################################
# version : birdben/logstash:v1
# desc : 当前版本安装的logstash
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 LOGSTASH 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV LOGSTASH_HOME /software/logstash-1.5.4

# 复制 logstash-1.5.4 文件到镜像中（logstash-1.5.4文件夹要和Dockerfile文件在同一路径）
ADD logstash-1.5.4 /software/logstash-1.5.4

# 解决环境问题，否则logstash无法从log文件中采集日志。具体环境： Logstash 1.5, Ubuntu 14.04, Oracle JDK7
RUN ln -s /lib/x86_64-linux-gnu/libcrypt.so.1 /usr/lib/x86_64-linux-gnu/libcrypt.so

# 挂载/logstash目录
VOLUME ["/logstash"]

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### logstash的supervisord.conf配置文件

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:logstash]
# 指定配置文件时，一定要使用绝对路径，相对路径是不好用的，这个坑已经踩过两次了。。
command=/software/logstash-1.5.4/bin/logstash -f /logstash/logstash.conf
```

##### logstash的控制台终端
```
# 构建镜像
$ docker build -t="birdben/logstash:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -t -i -v /docker/logstash:/logstash "birdben/logstash:v1"
```

##### kibana的Dockerfile文件
```
############################################
# version : birdben/kibana:v1
# desc : 当前版本安装的kibana
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 KIBANA 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV KIBANA_HOME /software/kibana-4.1.2

# 复制 kibana-4.1.2 文件到镜像中（kibana-4.1.2文件夹要和Dockerfile文件在同一路径）
ADD kibana-4.1.2 /software/kibana-4.1.2

# 容器需要开放Kibana的5601端口
EXPOSE 5601

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### kibana的supervisord.conf配置文件

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:kibana]
command=/software/kibana-4.1.2/bin/kibana
```

##### kibana的控制台终端
```
# 构建镜像
$ docker build -t="birdben/kibana:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 5601:5601 -t -i "birdben/kibana:v1"
```

##### 注意

ES和Redis如之前的文章介绍部署即可

##### 验证日志收集

> 访问Nginx主页，查看access.log中的日志是否按照指定的日志格式输出，之后access.log会被log_agent收集到Redis中，如果log_indexer是可用状态的，就会从Redis中消费掉日志信息，如果log_indexer是不可用的，日志信息就会保存在Redis中。log_indexer从Redis中消费掉日志后，就会在ES中创建相应地索引，最后在Kibana中进行查询显示


参考文章：

- http://www.cnblogs.com/yjf512/p/4568190.html
- http://www.cnblogs.com/yjf512/p/4199105.html