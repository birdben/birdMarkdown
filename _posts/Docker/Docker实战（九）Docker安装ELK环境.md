---
title: "Docker实战（九）Docker安装ELK环境"
date: 2015-12-24 04:19:47
tags: [Docker命令, Dockerfile, ELK]
categories: [Docker]
---

ELK实际上就是ElasticSearch，Logstash，Kibana的缩写，是日志收集分析的一种解决方案。

- Elasticsearch一个开源的搜索引擎框架（支持群集架构方式）
- Logstash集成各种收集日志插件，还是一个比较优秀的正则切割日志工具
- Kibana一个免费的web应用，支持在web端查看ES的搜索结果

elk是目前比较新也发展比较快的一套数据分析套件，其中Elasticsearch是用来作为存储和查询引擎的，kibana则是位于其之上的一个UI（更偏向于聚合汇总分析），而logstash则是属于ETL工具（数据的提取转换插入）。
在具体的使用过程中，目前觉得logstash算是比较鸡肋的，因为适用的场景有限，而且要扩展必须自己实现。个人建议，如果对es比较熟悉的，完全可以不需要用这个。自己用es加个river插件，那个效果也不错。

ELK简单架构
![ELK_Simple](http://img.blog.csdn.net/20151224033735994?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

日志收集系统架构整体架构
![日志收集系统架构](http://img.blog.csdn.net/20160103152529612?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

简单来讲他具体的工作流程就是Logstash agent监控并过滤日志，将过滤后的日志内容发给redis(这里的redis只处理队列不做存储)，Logstash index将日志收集在一起交给全文搜索服务ElasticSearch，可以用ElasticSearch进行自定义搜索，通过Kibana来结合 自定义搜索进行页面展示

此外 logstash 的收集方式分为 standalone 和 centralized。
standalone 是所有功能都在一个服务器上面，自发自收，centralized 就是集中收集，一台服务器接收所有shipper(个人理解就是logstash agent)的日志。
其实 logstash本身不分 什么 shipper 和 collector ，只不过就是配置文件不同而已，我们这次按照集中的方式来测试


这里的Logstash分为index和agent两种角色，也可以说是收集方式分为standalone和centralized两种。standalone是所有功能都在一个服务器上面，自发自收，centralized就是集中收集，一台服务器接收所有shipper(个人理解就是logstash agent)的日志。（其实logstash本身不分什么shipper和collector ，只不过就是配置文件不同而已）。Logstash的agent和indexer分开部署，多台agent负责监控、过滤日志，index负责收集日志并将日志交给ElasticSearch做搜索，通过Kibana来结合自定义搜索进行页面展示。Redis实际上是起到了缓冲消峰的作用，否则并发访问量大的时候ES会被拖垮的。使用redis的push和pop做队列，然后有个logstash_indexer来从队列中pop数据分析插入elasticsearch。这样做的好处是可扩展，logstash_agent只需要收集log进入队列即可，比较可能会有瓶颈的log分析使用logstash_indexer来做，而这个logstash_indexer又是可以水平扩展的，我可以在单独的机器上跑多个indexer来进行日志分析存储。

192.168.0.1 logstash index，ElasticSearch，kibana，JDK
192.168.0.2 logstash agent，JDK
192.168.0.3 redis

因为上一篇文章已经写过Docker如何安装ES环境了，这里我们直接继承上一次安装好的Docker镜像，所以重点只介绍Logstash和Kibana的安装，本文只是简单的单机ELK环境，后续会逐步完善ELK+Redis的环境，甚至会把ELK单独拆开三个Docker镜像使用

##### 安装Logstash（本文使用的是logstash的1.5.4版本）

```
# 下载Logstash
$ curl -O https://download.elastic.co/logstash/logstash/logstash-1.5.4.tar.gz

# 解压ES压缩包
$ tar -zxvf logstash-1.5.4.tar.gz

# 在{LOGSTASH_HOME}下新建一个conf目录，在里面新建一个配置文件logstash.conf 
$ cd logstash-1.5.4
$ mkdir conf
$ cd conf

# 编辑logstash.conf如下面的配置
$ vi logstash.conf

# 启动logstash
$ cd {LOGSTASH_HOME}/bin/
$ ./logstash -f ../conf/logstash.conf
```

##### 注意
因为java的默认heap size,回收机制等原因,logstash从1.4.0开始不再使用jar运行方式.

- 以前方式:
java -jar logstash-1.3.3-flatjar.jar agent -f logstash.conf
- 现在方式:
bin/logstash -f logstash.conf

##### logstash.conf配置文件

```
input {
	# 来自控制台	stdin {       type => "web"			# ES索引的type		codec => "json"			# 输入格式是json	}
	# 来自文件	file {
		# 文件所在的绝对路径		path => "/software/logstash-1.5.4/test.log"
		# ES索引的type		type => "system"
		# 文件格式是json		codec => "json"
		# 从文件的什么位置开始采集       start_position => "beginning"	}}output {
	# 输出到控制台	stdout {		codec => rubydebug	}
	# 输出到ES	elasticsearch {
		# 不使用logstash内嵌的ES		embedded => false		codec => "json"		protocol => "http"		host => "10.211.55.4"		port => 9200
		# 指定创建的索引名称		index => "birdlogstash"	}}
```

##### test.log文件内容，放在{LOGSTASH_HOME}目录下
```bird hello
bird testbird bye
```

##### 安装Kibana（本文使用的是kibana的4.1.2版本）

```
# 下载Kibana
$ curl -O https://download.elastic.co/kibana/kibana/kibana-4.1.2-linux-x64.tar.gz

# 解压ES压缩包
$ tar -zxvf kibana-4.1.2-linux-x64.tar.gz

# 重命名一下
$ mv kibana-4.1.2-linux-x64 kibana-4.1.2

# 启动Kibana
$ cd {KIBANA_HOME}/bin
$ ./kibana

# 访问http://192.168.1.120:5601/ 配置一个ElasticSearch索引 
# 在logstach里面添加数据 

```

##### 注意
```
如果Kibana和ES不在同一台机器上，需要在kibana.yml文件中指定ES集群的地址
# The Elasticsearch instance to use for all your queries.
elasticsearch_url: "http://10.211.55.4:9200"
```

##### Dockerfile文件
```
############################################
# version : birdben/elk:v1
# desc : 当前版本安装的elk
############################################
# 设置继承自我们创建的 elasticsearch 镜像
FROM birdben/elasticsearch:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN echo "export LC_ALL=C"

# 设置 ES 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV LOGSTASH_HOME /software/logstash-1.5.4
ENV KIBANA_HOME /software/kibana-4.1.2

# 复制 logstash-1.5.4, kibana-4.1.2 文件到镜像中（logstash-1.5.4, kibana-4.1.2文件夹要和Dockerfile文件在同一路径）
ADD logstash-1.5.4 /software/logstash-1.5.4
ADD kibana-4.1.2 /software/kibana-4.1.2

# 解决环境问题，否则logstash无法从log文件中采集日志。具体环境： Logstash 1.5, Ubuntu 14.04, Oracle JDK7
RUN ln -s /lib/x86_64-linux-gnu/libcrypt.so.1 /usr/lib/x86_64-linux-gnu/libcrypt.so

# 挂载/logstash目录
VOLUME ["/logstash"]

# 容器需要开放Kibana的5601端口
EXPOSE 5601

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：
https://github.com/birdben/birdDocker/blob/master/elk/Dockerfile

##### supervisor配置文件内容
```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:elasticsearch]
command=/bin/bash -c "exec ${ES_HOME}/bin/elasticsearch -DFOREGROUND"

[program:logstash]
# 指定配置文件时，一定要使用绝对路径，相对路径是不好用的，这个坑已经踩过两次了。。
command=/software/logstash-1.5.4/bin/logstash -f /logstash/logstash.conf

[program:kibana]
command=/software/kibana-4.1.2/bin/kibana
```

##### 注意
```
# 之前一直在supervisor使用如下配置来启动logstash，但是发现logstash刚启动起来自己就挂了，然后不断的在尝试重启。后来发现是配置文件没有找到，因为使用supervisor来配置服务的命令时，指定配置文件时，一定要使用绝对路径，相对路径是不好用的，这个坑已经踩过两次了。。这里再次鄙视一下自己。。

INFO success: logstash entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
INFO exited: logstash (exit status 1; not expected)
INFO spawned: 'logstash' with pid 12
INFO exited: logstash (exit status 1; not expected)
INFO spawned: 'logstash' with pid 13
INFO exited: logstash (exit status 1; not expected)
INFO gave up: logstash entered FATAL state, too many start retries too quickly

# 这里使用supervisorctl status查看supervisor监控的所有服务，就会发现ES没有处于被监控状态
$ supervisorctl status
logstash                    	   FATAL      Exited too quickly (process log may have details)
sshd                             RUNNING    pid 6, uptime 0:01:49
elasticsearch                    RUNNING    pid 8, uptime 0:01:49
```


##### 控制台终端

```
# 构建镜像
$ docker build -t="birdben/elk:v1" .
# 执行已经构件好的镜像
$ docker run -p 9999:22 -p 9200:9200 -p 9300:9300 -p 5601:5601 -t -i -v /docker/logstash:/logstash "birdben/elk:v1"
```

##### 测试Logstash

```
# 这里我们要测试几种方式的logstash输入和输出
logstash启动的参数
-e：代表控制台以字符串的方式输入conf配置
-f：代表指定文件的方式conf配置

############## logstash从控制台读取 ##############

# 只在CMD启动的进程export设置变量，而不是将变量赋值命令写入/etc/profile等脚本里，因此通过ssh方式登录容器获得的shell是没有这个变量的，所以ssh登录要提前设置JAVA_HOME环境变量
$ export JAVA_HOME=/software/jdk7

# 测试运行前端输出
$ {LOGSTASH_HOME}/bin/logstash -e 'input { stdin { } } output { stdout {} }'
Logstash startup completed
hello
2015-12-20T08:17:06.312Z c7f05b587d11 hello

# 也可以使用rubydebug的形式输出到控制台
$ {LOGSTASH_HOME}/bin/logstash -e 'input { stdin { } } output { stdout {codec=>rubydebug} }'
Logstash startup completed
hello
{
       "message" => "hello",
      "@version" => "1",
    "@timestamp" => "2015-12-20T13:35:38.996Z",
          "host" => "40421a32fbc5"
}

# 还可以将控制台的输入，输出到ES并且创建对应的索引
$ {LOGSTASH_HOME}/bin/logstash -e 'input { stdin { type => "web" codec => "json" } } output { stdout { codec => rubydebug } elasticsearch { embedded => false codec => "json" protocol => "http" host => "10.211.55.4" port => 9200 } }'
Logstash startup completed
{"name":"bird"}
{
          "name" => "bird",
      "@version" => "1",
    "@timestamp" => "2015-12-20T13:35:38.996Z",
          "type" => "web",
          "host" => "40421a32fbc5"
}

# 执行之后，可以查询ES的索引，会自动创建一个logstash的索引，并且会有一个对应的属性name，它的值是bird
$ curl -XPOST 'http://10.211.55.4:9200/_search?pretty' -d '{"query":{"match_all":{}}}'

############## logstash从文件中读取 ##############

$ {LOGSTASH_HOME}/bin/logstash -f /logstash/logstash.conf

# 从文件中读取日志，可能会遇到下面的问题
NotImplementedError: block device detection unsupported or native support failed to load
    from org/jruby/RubyFileTest.java:67:in `blockdev?'
    from (irb):1:in `evaluate'
    from org/jruby/RubyKernel.java:1107:in `eval'
    from org/jruby/RubyKernel.java:1507:in `loop'
    from org/jruby/RubyKernel.java:1270:in `catch'
    from org/jruby/RubyKernel.java:1270:in `catch'
    from /home/ubuntu/logstash-1.5.0-rc3/lib/logstash/runner.rb:77:in `run'
    from org/jruby/RubyProc.java:271:in `call'
    from /home/ubuntu/logstash-1.5.0-rc3/lib/logstash/runner.rb:131:in `run'
    from org/jruby/RubyProc.java:271:in `call'
    from /home/ubuntu/logstash-1.5.0-rc3/vendor/bundle/jruby/1.9/gems/stud-0.0.19/lib/stud/task.rb:12:in `initialize'

# 上面的问题原因是环境问题，解决方案是先执行下面的语句，然后在运行logstash
ln -s /lib/x86_64-linux-gnu/libcrypt.so.1 /usr/lib/x86_64-linux-gnu/libcrypt.so

# 参考文章：
https://github.com/elastic/logstash/issues/3127#issuecomment-101068714

# 改好上面的问题之后logstash就会将文件的内容读取并输出到ES，使用下面的语句进行查询，就可以看到之前test.log中的3行记录，被ES创建了3条索引记录
$ curl -XPOST 'http://10.211.55.4:9200/_search?pretty' -d '{"query":{"match_all":{}}}'
```

##### 测试Kibana
```
# 浏览器直接访问
http://10.211.55.4:5601

# 如果ES还没有索引，你需要告诉它你打算探索哪个 Elasticsearch 索引。第一次访问 Kibana 的时候，你会被要求定义一个 index pattern 用来匹配一个或者多个索引名。好了。这就是你需要做的全部工作。以后你还可以随时从 Settings 标签页添加更多的 index pattern。

# 因为我们在Logstash配置了从log文件中读取数据并且输出到ES的索引上，配置文件中已经指定了索引的名称"birdlogstash"，这样我们在Kibana只要指定这个索引名称就可以了，同理我们也可以在Logstash中改成按照日期分割的方式，Kibana也可以按照这种方式来配置。

# 指定创建的索引名称index => "birdlogstash"

# 指定创建的索引名称（按照索引类型和日期分割）index => "logstash-%{type}-%{+YYYY.MM.dd}"

# 默认情况下，Kibana 会连接运行在 localhost 的 Elasticsearch。要连接其他 Elasticsearch 实例，修改 kibana.yml 里的 Elasticsearch URL，然后重启 Kibana。如何在生产环境下使用 Kibana，阅读生产环境部署章节。
http://kibana.logstash.es/content/kibana/v4/production.html

```


参考文章：

- http://www.iyunv.com/thread-73371-1-1.html
- http://kibana.logstash.es/content/logstash/plugins/output/elasticsearch.html
- http://langke.name/2014/12/18/elk-2.html
- http://blog.chinaunix.net/uid-532511-id-4836683.html
- http://www.wklken.me/posts/2015/04/26/elk-for-nginx-log.html#3-supervisor
- http://ju.outofmemory.cn/entry/108925
- http://article.yeeyan.org/view/536432/459461
- http://qindongliang.iteye.com/blog/2250776www