---
title: "Flume学习（七）Flume整合Elasticsearch2.x"
date: 2016-08-27 16:17:39
tags: [Flume]
categories: [Log]
---

### 环境简介

- JDK1.7.0_79
- Flume1.6.0
- Elasticsearch2.0.0

### Flume不支持Elasticsearch2.x版本

目前官方Flume最新的版本是1.6.0，该版本只支持Elasticsearch1.7.x的版本，暂时不支持Elasticsearch2.x版本，因为Elasticsearch2.x版本做了比较大的改动，很多API都已经废弃不用了，具体可以参考下面的连接

- https://github.com/elastic/elasticsearch/issues/14187

### 第三方ElasticsearchSink2支持2.x版本

这里我找到了一个第三方开源的FlumeSink插件来支持Elasticsearch2.x版本

- https://github.com/lucidfrontier45/ElasticsearchSink2

但是这个项目是使用Gradle编译打包的，所以下面先简单介绍下Gradle的安装和使用

我这里使用的Mac，所以安装Gradle很简单。但是Gradle是依赖于JVM的

```
$ brew install gradle
$ gradle -v
```

然后从github下载ElasticsearchSink2的源代码，并且导入到idea中，然后执行下面gradle命令构建项目（下面这两个脚本是ElasticsearchSink2项目自带的），会在ElasticsearchSink2/build/libs/目录下生成对应的jar包

```
# 构建标准的jar包
$ ./gradlew build

# 构建包含Elasticsearch依赖的jar包
$ ./gradlew assembly
```

添加刚才构建好的elasticsearch-sink2-1.0.jar到Flume的classpath或者是Flume的lib目录，删除Flume的lib目录下的guava-*.jar，jackson-core-*.jar。

### 具体的配置文件
#### Flume Agent端的flume_es2.conf配置

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-es-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 127.0.0.1
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8
agentX.sources.flume-avro-sink.interceptors = es_interceptor
agentX.sources.flume-avro-sink.interceptors.es_interceptor.type = regex_extractor
#agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (\"([^,^\"]+)\":\"([^:^\"]+)\")|(\"([^,^\"]+)\":([\\d]+))
#agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (\\d):(\\d):(\\d):(\\d):(\\d):(\\d)

# mapping不正确没有匹配成功
#agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (TIME:(.*?)),(HOSTNAME:(.*?)),(LI:(.*?)),(LU:(.*?)),(NU:(.*?)),(CMD:(.*?))
# mapping正确，数据匹配不正确包含了多余的字段名
#agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (\"TIME\":(.*?)),(\"HOSTNAME\":(.*?)),(\"LI\":(.*?)),(\"LU\":(.*?)),(\"NU\":(.*?)),(\"CMD\":(.*?))
# mapping正确，数据也正确（{}需要转义，转义符是\\）
agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = "TIME":(.*?),"HOSTNAME":(.*?),"LI":(.*?),"LU":(.*?),"NU":(.*?),"CMD":(.*?)

agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers = s1 s2 s3 s4 s5 s6
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s1.name = aaa
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s2.name = bbb
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s3.name = s3
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s4.name = s4
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s5.name = s5
agentX.sources.flume-avro-sink.interceptors.es_interceptor.serializers.s6.name = s6

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

agentX.sinks.flume-es-sink.channel = chX
agentX.sinks.flume-es-sink.type = com.frontier45.flume.sink.elasticsearch2.ElasticSearchSink
# 每个事务写入多少个Event
agentX.sinks.flume-es-sink.batchSize = 100
agentX.sinks.flume-es-sink.hostNames = 127.0.0.1:9300
# 注意：indexName必须小写
agentX.sinks.flume-es-sink.indexName = command_index
agentX.sinks.flume-es-sink.indexType = logs
agentX.sinks.flume-es-sink.clusterName = ben-es
# ttl 的时间，过期了会自动删除文档，如果没有设置则永不过期，ttl使用integer或long型，单位可以是：ms (毫秒), s (秒), m (分), h (小时), d (天) and w (周)。例如：a1.sinks.k1.ttl = 5d则表示5天后过期。这里没用到
# agentX.sinks.flume-es-sink.ttl = 5d
agentX.sinks.flume-es-sink.serializer = com.frontier45.flume.sink.elasticsearch2.ElasticSearchLogStashEventSerializer
```

#### flume_es2.conf这里的配置需要修改

```
sink.type = com.frontier45.flume.sink.elasticsearch2.ElasticSearchSink
sink.serializer = com.frontier45.flume.sink.elasticsearch2.ElasticSearchLogStashEventSerializer
```

#### Avro Cient端的flume.conf配置

```
agent3.sources = command-logfile-source
agent3.channels = ch3
agent3.sinks = flume-avro-sink

agent3.sources.command-logfile-source.channels = ch3
agent3.sources.command-logfile-source.type = exec
agent3.sources.command-logfile-source.command = tail -F /Users/yunyu/Downloads/command.log

agent3.channels.ch3.type = memory
agent3.channels.ch3.capacity = 1000
agent3.channels.ch3.transactionCapacity = 100

agent3.sinks.flume-avro-sink.channel = ch3
agent3.sinks.flume-avro-sink.type = avro
agent3.sinks.flume-avro-sink.hostname = 127.0.0.1
agent3.sinks.flume-avro-sink.port = 41414
```

#### 启动ES服务

```
$ ./bin/elasticseach -d
```

#### 启动Flume Agent端

```
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_es2.conf -Dflume.root.logger=DEBUG,console -n agentX
```

#### 启动Flume AvroCliet端

```
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_es2.conf -Dflume.root.logger=DEBUG,console -n agentX
```

启动Flume Agent可能会遇到如下的错误：

```
2016-08-27 14:25:58,045 (lifecycleSupervisor-1-1) [ERROR - org.apache.flume.lifecycle.LifecycleSupervisor$MonitorRunnable.run(LifecycleSupervisor.java:253)] Unable to start SinkRunner: { policy:org.apache.flume.sink.DefaultSinkProcessor@76d45adb counterGroup:{ name:null counters:{} } } - Exception follows.
java.lang.NoSuchFieldError: LUCENE_5_3_1
	at org.elasticsearch.Version.<clinit>(Version.java:279)
	at org.elasticsearch.client.transport.TransportClient$Builder.build(TransportClient.java:131)
	at com.frontier45.flume.sink.elasticsearch2.client.ElasticSearchTransportClient.openClient(ElasticSearchTransportClient.java:198)
```

出现上面的错误是因为lucene的版本不对，这里我开始尝试安装的是下面的几个版本：

- Elasticsearch-2.2.2：没有选择Elasticsearch-2.2.2版本是因为ik分词插件没有对应的版本
- Elasticsearch-2.2.1：没有选择Elasticsearch-2.2.1版本是因为报错没有找到LUCENE_5_3_1对应的Lucene-5.3.1版本，但是这里下载Elasticsearch-2.2.1版本中的jar包依赖的是Lucene-5.4.1版本，所以找不到Lucene-5.3.1版本。这里可能是因为ElasticsearchSink2的问题，暂时先换成ElasticsearchSink2使用的2.0.0版本，后续在尝试单独升级Lucene-5.4.1试试。
- Elasticsearch-2.0.0：ElasticsearchSink2使用的此版本

检查一下ES源码的pom文件就可以知道ES和Lucene的版本对应关系如下：

ES版本						|Lucene版本		|ik插件版本
------						|------			|------
Elasticsearch-2.2.2		|Lucene-5.4.1		|无
Elasticsearch-2.2.1		|Lucene-5.4.1		|1.8.1
Elasticsearch-2.0.0		|Lucene-5.2.1		|1.5.0

#### 检查ES的索引数据

如下图所示，ES的mapping和索引数据都正确，说明我们使用ElasticsearchSink2的方式成功将Flume1.6.0采集的command.log日志数据写入到Elasticsearch2.0.0版本里

![mapping](http://img.blog.csdn.net/20160827161437043?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![data](http://img.blog.csdn.net/20160827161523009?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

参考文章：

- https://github.com/lucidfrontier45/ElasticsearchSink2
- https://github.com/elastic/elasticsearch/issues/14187