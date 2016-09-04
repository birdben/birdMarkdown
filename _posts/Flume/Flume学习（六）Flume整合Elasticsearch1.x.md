---
title: "Flume学习（六）Flume整合Elasticsearch1.x"
date: 2016-08-26 23:11:56
tags: [Flume]
categories: [Log]
---

### 环境简介

- JDK1.7.0_79
- Flume1.6.0
- Elasticsearch1.7.3

### ES的安装和配置

这里不做重点介绍，请参考之前关于ES的文章

### Flume整合ES的相关配置

#### flume_es.conf配置文件

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-es-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 10.10.1.23
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

agentX.sinks.flume-es-sink.channel = chX
agentX.sinks.flume-es-sink.type = org.apache.flume.sink.elasticsearch.ElasticSearchSink
# 每个事务写入多少个Event
agentX.sinks.flume-es-sink.batchSize = 100
agentX.sinks.flume-es-sink.hostNames = 10.10.1.23:9300
# 注意：indexName必须小写
agentX.sinks.flume-es-sink.indexName = command_index
agentX.sinks.flume-es-sink.indexType = logs
agentX.sinks.flume-es-sink.clusterName = es
# ttl 的时间，过期了会自动删除文档，如果没有设置则永不过期，ttl使用integer或long型，单位可以是：ms (毫秒), s (秒), m (分), h (小时), d (天) and w (周)。例如：a1.sinks.k1.ttl = 5d则表示5天后过期。这里没用到
# agentX.sinks.flume-es-sink.ttl = 5d
agentX.sinks.flume-es-sink.serializer = org.apache.flume.sink.elasticsearch.ElasticSearchLogStashEventSerializer
```

然后我们先启动ES，然后再启动Flume来收集command.log日志并且写入到ES。

#### Flume启动报错

如果启动Flume的时候，报如下的错误，说明缺少ES相关依赖的jar包。需要将${ES\_HOME}/lib/lucene-core-4.10.4.jar，${ES\_HOME}/lib/elasticsearch-1.7.3.jar这两个包复制到${FLUME_HOME}/lib/下

```
2016-08-25 10:37:50,303 (conf-file-poller-0) [ERROR - org.apache.flume.node.PollingPropertiesFileConfigurationProvider$FileWatcherRunnable.run(PollingPropertiesFileConfigurationProvider.java:145)] Failed to start agent because dependencies were not found in classpath. Error follows.
java.lang.NoClassDefFoundError: org/elasticsearch/common/io/BytesStream
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:191)
	at org.apache.flume.sink.elasticsearch.ElasticSearchSink.configure(ElasticSearchSink.java:286)
	at org.apache.flume.conf.Configurables.configure(Configurables.java:41)
	at org.apache.flume.node.AbstractConfigurationProvider.loadSinks(AbstractConfigurationProvider.java:413)
	at org.apache.flume.node.AbstractConfigurationProvider.getConfiguration(AbstractConfigurationProvider.java:98)
	at org.apache.flume.node.PollingPropertiesFileConfigurationProvider$FileWatcherRunnable.run(PollingPropertiesFileConfigurationProvider.java:140)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:471)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:304)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:178)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.ClassNotFoundException: org.elasticsearch.common.io.BytesStream
	at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
	at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
	at java.security.AccessController.doPrivileged(Native Method)
	at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
	... 14 more
```

#### 日志解析问题

当command.log有日志不断输出时，我们会看到Flume控制台会不断收集到ES，但是在ES端查询command_index的mapping却和我们想象的mapping不太一样，这里我们需要解析command.log日志中的日志格式，将具体的日志字段解析出来并且对应ES mapping中的字段。我之前有用过ELK的方式来做command.log日志的收集，Logstash通过filter的grok表达式的方式来解析日志格式很简单，并且可以对收集的日志字段进行一些特殊处理（如：类型转换，删除字段，重命名字段等等）。在Flume里是通过Interceptors来实现Logstash的filter grok表达式功能的。

![ES_Mapping](http://img.blog.csdn.net/20160825104414733?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Flume的Interceptors配置

下面是我们command.log的日志文件截取，可以看出我们command.log日志的格式比较简单。修改我们的flume_es.conf配置文件，添加interceptors的配置，指定正则表达式解析我们的日志格式。

##### command.log日志文件

```
  598  {"TIME":"2016-08-24 19:07:49","HOSTNAME":"localhost","LI":"8844","LU":"yunyu","NU":"yunyu","CMD":"java -version"}
  598  {"TIME":"2016-08-24 19:07:49","HOSTNAME":"localhost","LI":"8844","LU":"yunyu","NU":"yunyu","CMD":"java -version"}
  599  {"TIME":"2016-08-24 19:15:19","HOSTNAME":"localhost","LI":"8844","LU":"yunyu","NU":"yunyu","CMD":"cd ~/dev/elasticsearch-1.7.3/config/"}
  600  {"TIME":"2016-08-24 19:15:21","HOSTNAME":"localhost","LI":"8844","LU":"yunyu","NU":"yunyu","CMD":"sublime elasticsearch.yml "}
  515  {"TIME":"2016-08-25 10:00:07","HOSTNAME":"localhost","LI":"6601","LU":"yunyu","NU":"yunyu","CMD":"ls"}
  515  {"TIME":"2016-08-25 10:00:07","HOSTNAME":"localhost","LI":"6601","LU":"yunyu","NU":"yunyu","CMD":"ls"}
```

我擦，我被Flume的interceptors配置给坑了，官网给了一个interceptors的例子非常的简单，根本就不知道怎么支持我上面的日志格式（主要是我正则表达式学的太烂了）。感觉Flume对于日志的处理方面没有Logstash灵活易用。

先来看一下官网给的interceptors例子吧

If the Flume event body contained 1:2:3.4foobar5 and the following configuration was used

```
# ()号中的是从日志记录中提取出来的value，这个value会对应serializers中定义的field名称
a1.sources.r1.interceptors.i1.regex = (\\d):(\\d):(\\d)
a1.sources.r1.interceptors.i1.serializers = s1 s2 s3
a1.sources.r1.interceptors.i1.serializers.s1.name = one
a1.sources.r1.interceptors.i1.serializers.s2.name = two
a1.sources.r1.interceptors.i1.serializers.s3.name = three
```
The extracted event will contain the same body but the following headers will have been added one=>1, two=>2, three=>3

上面官网的例子就是从日志1:2:3.4foobar5中按照我们的格式用":"号分割并且提取后面的1位数字，并且按照提取出来的值顺序对应字段名称为：one，two，three，最后转换出来的结果就是one=>1, two=>2, three=>3

但是我们上面的日志文件格式要稍微复杂一点，我们的日志一个json格式的字符串，所以第一次用Flume提取日志记录中的值还有点费事，我也是一边修改正则表达式，一边测试结果。

##### mapping不正确没有匹配成功

```
agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (TIME:(.*?)),(HOSTNAME:(.*?)),(LI:(.*?)),(LU:(.*?)),(NU:(.*?)),(CMD:(.*?))
```

![mapping](http://img.blog.csdn.net/20160825172129720?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![data](http://img.blog.csdn.net/20160825172217307?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### mapping正确，数据匹配不正确包含了多余的字段名

```
agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = (\"TIME\":(.*?)),(\"HOSTNAME\":(.*?)),(\"LI\":(.*?)),(\"LU\":(.*?)),(\"NU\":(.*?)),(\"CMD\":(.*?))
```

![mapping](http://img.blog.csdn.net/20160825172611373?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![data](http://img.blog.csdn.net/20160825172633442?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### mapping正确，数据也正确（特殊字符需要转义，转义符是\\）

最后终于调试正确了，是因为我修改Flume的正则表达式改错了，发现es.log中的错误信息提示的bulk参数，我发现bulk的参数居然解析了两次时间字段{"TIME":"2016-08-25 16:09:10"}，我们定义的aaa字段包含了TIME字段名还有value，但是bbb字段却只有TIME字段的value，而不是预想中第二个字段的值。所以这里我发现了一个很重要的规则，就是正则表达式中"()号"中的是从日志记录中提取出来的value，这个value会对应serializers中定义的field名称，我上一个表达式之所以会解析两边TIME字段就是因为正则表达式中带了两个"()号"，如：(TIME:(.*?))，这样就会把TIME的value提取出来一次，再把TIME:value这样的字符串当成值提取出来一次，所以就会出现上面的情况。

```
failed to execute bulk item (index) index {[command_index-2016-08-25][logs][AVbAvyBwi0A7kguFbmpj], source[{"@message":537,"@fields":{"aaa":{"TIME":"2016-08-25 16:09:10"},"aaa":"{\"TIME\":\"2016-08-25 16:09:10\"","s5":"\"LI\":\"5573\"","s6":"\"5573\"","bbb":"\"2016-08-25 16:09:10\"","s3":"\"HOSTNAME\":\"localhost\"","s4":"\"localhost\""}}]}
```

```
agentX.sources.flume-avro-sink.interceptors.es_interceptor.regex = "TIME":(.*?),"HOSTNAME":(.*?),"LI":(.*?),"LU":(.*?),"NU":(.*?),"CMD":(.*?)
```

![mapping](http://img.blog.csdn.net/20160825164926247?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![data](http://img.blog.csdn.net/20160825164958191?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


##### flume_es.conf配置文件

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-es-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 10.10.1.23
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
agentX.sinks.flume-es-sink.type = org.apache.flume.sink.elasticsearch.ElasticSearchSink
# 每个事务写入多少个Event
agentX.sinks.flume-es-sink.batchSize = 100
agentX.sinks.flume-es-sink.hostNames = 10.10.1.23:9300
# 注意：indexName必须小写
agentX.sinks.flume-es-sink.indexName = command_index
agentX.sinks.flume-es-sink.indexType = logs
agentX.sinks.flume-es-sink.clusterName = es
# ttl 的时间，过期了会自动删除文档，如果没有设置则永不过期，ttl使用integer或long型，单位可以是：ms (毫秒), s (秒), m (分), h (小时), d (天) and w (周)。例如：a1.sinks.k1.ttl = 5d则表示5天后过期。这里没用到
# agentX.sinks.flume-es-sink.ttl = 5d
agentX.sinks.flume-es-sink.serializer = org.apache.flume.sink.elasticsearch.ElasticSearchLogStashEventSerializer
```

参考文章：

- http://blog.csdn.net/yangruihong/article/details/17245359
- http://flume.apache.org/FlumeUserGuide.html#zookeeper-based-configuration
- http://blog.csdn.net/xiao_jun_0820/article/details/38333171
- http://www.voidcn.com/blog/lanmo555/article/p-4974586.html
- http://blog.csdn.net/sunflower_cao/article/details/39929931
- http://blog.csdn.net/u010022051/article/details/50515725