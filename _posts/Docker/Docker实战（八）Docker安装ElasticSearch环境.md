---
title: "Docker实战（八）Docker安装ElasticSearch环境"
date: 2015-12-20 00:56:34
tags: [Docker命令, Dockerfile, ElasticSearch]
categories: [Docker]
---

基本步骤和之前几篇文章一样，请参考前面的相关文章

##### ElasticSearch安装

- 1.安装ES
- 2.安装head,bigdesk插件
- 3.安装ik插件
- 4.配置ES集群

##### 安装ES（本文使用的是elasticsearch的1.7.2版本）

```
# 下载ES
$ curl -O https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.2.tar.gz

# 解压ES压缩包
$ tar -zxvf elasticsearch-1.7.2.tar.gz

# 启动ES
$ sudo ./elasticsearch -d

```

##### 安装head,bigdesk,HQ,kopf插件（可选择安装，建议安装head和bigdesk）

```
$ ${ES_HOME}/bin/plugin --install mobz/elasticsearch-head
# 安装完成访问：http://localhost:9200/_plugin/head/

$ ${ES_HOME}/bin/plugin --install lukas-vlcek/bigdesk
# 安装完成访问：http://localhost:9200/_plugin/bigdesk/#nodes

$ ${ES_HOME}/bin/plugin -install royrusso/elasticsearch-HQ
# 安装完成访问：http://localhost:9200/_plugin/HQ/

$ ${ES_HOME}/bin/plugin -install lmenezes/elasticsearch-kopf
# 安装完成访问：http://localhost:9200/_plugin/kopf/#!/cluster
```

##### 安装ik插件

```
# 因为我使用的是老版本的ES，所以ik插件也使用的是对应老版本的
$ git clone https://github.com/medcl/elasticsearch-analysis-ik
$ cd elasticsearch-analysis-ik
$ mvn clean
$ mvn compile
$ mvn package
$ cd target
$ cp elasticsearch-analysis-ik-xxx.jar ${ES_HOME}/plugins/ik/

$ cd elasticsearch-analysis-ik
$ cp config/ik ${ES_HOME}/config/
```

##### 配置elasticsearch.yml文件，在文件的最后添加下面的配置

```
index:
  analysis:
    analyzer:
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true

index.analysis.analyzer.default.type: ik
index.store.type: niofs
```

##### 设置ES的内存大小
```
$ {ES_HOME}/bin
$ vi elasticsearch

# 设置ES的最大内存，最小内存
ES_MIN_MEM=4g
ES_MAX_MEM=4g
```
##### 注意
如果这里不设置ES的内存大小，后面整合ELK环境的时候，Logstash会无法批量在ES创建索引而报错，具体错误信息如下。所以在ES启动时设置ES的内存大小，建议是实际内存的一半

```
Got error to send bulk of actions: Connection reset {:level=>:error}
```

这个内存配置不能随便设置，必须根据实际情况进行设置，否则设置之后ES就启动不起来了，报内存溢出的错误，建议尽量不要随意修改此配置

##### Dockerfile文件
```
############################################
# version : birdben/elasticsearch:v1
# desc : 当前版本安装的elasticsearch
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN echo "export LC_ALL=C"

# 设置 ES 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV ES_HOME /software/elasticsearch-1.7.2

# 复制 elasticsearch-1.7.2 文件到镜像中（elasticsearch-1.7.2文件夹要和Dockerfile文件在同一路径）
ADD elasticsearch-1.7.2 /software/elasticsearch-1.7.2

# 容器需要开放ES的9200和9300端口
EXPOSE 9200
EXPOSE 9300

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/elasticsearch/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

# 修改supervisor配置方式如下，修改为不自动重启ES，并且改成非daemon，DFOREGROUND的方式运行，supervisor就可以监控到了
[program:elasticsearch]
command=/bin/bash -c "exec ${ES_HOME}/bin/elasticsearch -DFOREGROUND"
```

##### 注意supervisor配置
```
# 之前一直在supervisor使用如下配置来启动ES，但是仔细观察Docker的控制台输出会发现如下的错误
[program:elasticsearch]
command=/{ES_HOME}/bin/elasticsearch -d

INFO success: elasticsearch entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
INFO exited: elasticsearch (exit status 1; not expected)
INFO spawned: 'elasticsearch' with pid 12
INFO exited: elasticsearch (exit status 1; not expected)
INFO spawned: 'elasticsearch' with pid 13
INFO exited: elasticsearch (exit status 1; not expected)
INFO gave up: elasticsearch entered FATAL state, too many start retries too quickly

# 这里使用supervisorctl status查看supervisor监控的所有服务，就会发现ES没有处于被监控状态
$ supervisorctl status
elasticsearch                    FATAL      Exited too quickly (process log may have details)
sshd                             RUNNING    pid 6, uptime 0:01:49
tomcat                           RUNNING    pid 8, uptime 0:01:49

# 所以这里修改supervisor配置方式如下，修改为不自动重启ES，并且改成非daemon，DFOREGROUND的方式运行，supervisor就可以监控到了
[program:elasticsearch]startsecs = 0autorestart = falsecommand=/bin/bash -c "exec ${ES_HOME}/bin/elasticsearch -DFOREGROUND"

# 这里说明一下supervisor启动多个服务，要求所有启动的服务都是非daemon的方式启动，否则就会遇到如上的问题，autorestart设置为false只是为了让supervisor启动报错的时候不会重复启动，只要改成非daemon的方式启动ES，可以设置autorestart为true
```

参考文章

- http://www.dedoimedo.com/computers/docker-supervisord.html
- http://dockerpool.com/static/books/docker_practice/cases/supervisor.html
- http://charlesleifer.com/blog/setting-up-elasticsearch-with-basic-auth-and-ssl-for-use-with-python/

##### 控制台终端

```
# 构建镜像
$ docker build -t="birdben/elasticsearch:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 9200:9200 -p 9300:9300 -t -i 'birdben/elasticsearch:v1'
```

##### 访问ElasticSearch的插件测试

```
http://10.211.55.4:9200/_plugin/head/
http://10.211.55.4:9200/_plugin/bigdesk/#nodes
```

##### User索引的mapping

```
{
  "user": {
    "dynamic" : "strict",
    "properties": {
      "id": {
        "type": "string",
        "index": "not_analyzed"
      },
      "name": {
        "type": "string",
        "index_analyzer": "ik",
        "search_analyzer": "ik_smart"
      },
      "age": {
        "type": "integer"
      },
      "job": {
        "type": "string",
        "index_analyzer": "ik",
        "search_analyzer": "ik_smart"
      },
      "createTime": {
        "type": "long"
      }
    }
  }
}
```

##### 新建索引测试ik分词

```
# 尝试创建user索引
$ curl -XPOST 'http://10.211.55.4:9200/user?pretty' -d '@user.json'

# 如果遇到下面的错误，原因是${ES_HOME}/lib/下需要引入httpclient-4.5.jar, httpcore-4.4.1.jar
{
  "error" : "IndexCreationException[[user] failed to create index]; nested: NoClassDefFoundError[org/apache/http/client/ClientProtocolException]; nested: ClassNotFoundException[org.apache.http.client.ClientProtocolException]; ",
  "status" : 500
}

# 创建索引成功后，查看索引信息
$ curl -XGET 'http://10.211.55.4:9200/_cat/indices?pretty'
green open user 5 1 0 0 970b 575b

# 测试ik分词效果
$ curl -XGET 'http://10.211.55.4:9200/goods/_analyze?analyzer=ik&pretty=true' -d '{"text":"世界如此之大"}'

# 测试ik_smart分词效果
$ curl -XGET 'http://10.211.55.4:9200/user/_analyze?analyzer=ik_smart&pretty=true' -d '{"text":"世界如此之大"}'
```

参考文章：

- https://github.com/medcl/elasticsearch-analysis-ik
- http://my.oschina.net/xiaohui249/blog/232784
- http://my.oschina.net/xiaohui249/blog/228748
- http://www.mamicode.com/info-detail-943587.html
- http://ask.csdn.net/questions/154112

