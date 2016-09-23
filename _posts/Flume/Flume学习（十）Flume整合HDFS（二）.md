---
title: "Flume学习（十）Flume整合HDFS（二）"
date: 2016-09-23 14:09:26
tags: [Flume, HDFS]
categories: [Log]
---

上一篇介绍了Flume整合HDFS，但是没有对HDFS Sink进行配置上的优化，本篇重点介绍HDFS Sink的相关配置。

上一篇中我们用Flume采集的日志直接输出到HDFS文件中，但是文件的输出的文件大小

#### 优化后的flume\_collector\_hdfs.conf配置文件

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-hdfs-sink

agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 127.0.0.1
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8

# 定义拦截器，为消息添加时间戳和Host地址
agentX.sources.flume-avro-sink.interceptors = i1 i2
agentX.sources.flume-avro-sink.interceptors.i1.type = timestamp
agentX.sources.flume-avro-sink.interceptors.i2.type = host
# 如果不指定hostHeader，就是用%{host}。但是指定了hostHeader=hostname，就需要使用%{hostname}
agentX.sources.flume-avro-sink.interceptors.i2.hostHeader = hostname
agentX.sources.flume-avro-sink.interceptors.i2.preserveExisting = true
agentX.sources.flume-avro-sink.interceptors.i2.useIP = true

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

agentX.sinks.flume-hdfs-sink.type = hdfs
agentX.sinks.flume-hdfs-sink.channel = chX

# agentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/
# 使用时间作为分割目录
agentX.sinks.flume-hdfs-sink.hdfs.path = hdfs://10.10.1.64:8020/flume/events/%Y%m%d/

# HdfsEventSink中，hdfs.fileType默认为SequenceFile，将其改为DataStream就可以按照采集的文件原样输入到hdfs，加一行agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream
# 设置文件格式， 有3种格式可选择：SequenceFile, DataStream or CompressedStream
# 当使用DataStream时候，文件不会被压缩，不需要设置hdfs.codeC
# 当使用CompressedStream时候，必须设置一个正确的hdfs.codeC值
agentX.sinks.flume-hdfs-sink.hdfs.fileType = DataStream

# 写入hdfs的文件名前缀，可以使用flume提供的日期及%{host}表达式。默认值：FlumeData
agentX.sinks.flume-hdfs-sink.hdfs.filePrefix = events-%{hostname}-
# 写入hdfs的文件名后缀，比如：.lzo .log等。
# agentX.sinks.flume-hdfs-sink.hdfs.fileSuffix = .log

# 临时文件的文件名前缀，hdfs sink会先往目标目录中写临时文件，再根据相关规则重命名成最终目标文件
# agentX.sinks.flume-hdfs-sink.hdfs.inUsePrefix
# 临时文件的文件名后缀。默认值：.tmp
# agentX.sinks.flume-hdfs-sink.hdfs.inUseSuffix

# 当目前被打开的临时文件在该参数指定的时间（秒）内，没有任何数据写入，则将该临时文件关闭并重命名成目标文件。默认值是0
# agentX.sinks.flume-hdfs-sink.hdfs.idleTimeout = 0
# 文件压缩格式，包括：gzip, bzip2, lzo, lzop, snappy
# agentX.sinks.flume-hdfs-sink.hdfs.codeC = gzip
# 每个批次刷新到HDFS上的events数量。默认值：100
# agentX.sinks.flume-hdfs-sink.hdfs.batchSize = 100

# 不想每次Flume将日志写入到HDFS文件中都分成很多个碎小的文件，这里控制HDFS的滚动
# 注：滚动（roll）指的是，hdfs sink将临时文件重命名成最终目标文件，并新打开一个临时文件来写入数据；
# 设置间隔多长将临时文件滚动成最终目标文件。单位是秒，默认30秒。
# 如果设置为0的话表示不根据时间滚动hdfs文件
agentX.sinks.flume-hdfs-sink.hdfs.rollInterval = 0
# 当临时文件达到该大小（单位：bytes）时，滚动成目标文件。默认值1024，单位是字节。
# 如果设置为0的话表示不基于文件大小滚动hdfs文件
agentX.sinks.flume-hdfs-sink.hdfs.rollSize = 0
# 设置当events数据达到该数量时候，将临时文件滚动成目标文件。默认值是10个。
# 如果设置为0的话表示不基于事件个数滚动hdfs文件
agentX.sinks.flume-hdfs-sink.hdfs.rollCount = 300

# 是否启用时间上的”舍弃”，这里的”舍弃”，类似于”四舍五入”，后面再介绍。如果启用，则会影响除了%t的其他所有时间表达式
# agentX.sinks.flume-hdfs-sink.hdfs.round = true
# 时间上进行“舍弃”的值。默认值：1
# 举例：当时间为2015-10-16 17:38:59时候，hdfs.path依然会被解析为：/flume/events/20151016/17:30/00
# 因为设置的是舍弃10分钟内的时间，因此，该目录每10分钟新生成一个。
# agentX.sinks.flume-hdfs-sink.hdfs.roundValue = 10
# 时间上进行”舍弃”的单位，包含：second,minute,hour。默认值：seconds
# agentX.sinks.flume-hdfs-sink.hdfs.roundUnit = minute

# 写入HDFS文件块的最小副本数。默认值：HDFS副本数
# agentX.sinks.flume-hdfs-sink.hdfs.minBlockReplicas
# 最大允许打开的HDFS文件数，当打开的文件数达到该值，最早打开的文件将会被关闭。默认值：5000
# agentX.sinks.flume-hdfs-sink.hdfs.maxOpenFiles
# 执行HDFS操作的超时时间（单位：毫秒）。默认值：10000
# agentX.sinks.flume-hdfs-sink.hdfs.callTimeout
# hdfs sink启动的操作HDFS的线程数。默认值：10
# agentX.sinks.flume-hdfs-sink.hdfs.threadsPoolSize
# 时区。默认值：Local Time
# agentX.sinks.flume-hdfs-sink.hdfs.timeZone
# 是否使用当地时间。默认值：flase
# agentX.sinks.flume-hdfs-sink.hdfs.useLocalTimeStamp
# hdfs sink关闭文件的尝试次数。默认值：0
# 如果设置为1，当一次关闭文件失败后，hdfs sink将不会再次尝试关闭文件，这个未关闭的文件将会一直留在那，并且是打开状态。
# 设置为0，当一次关闭失败后，hdfs sink会继续尝试下一次关闭，直到成功。
# agentX.sinks.flume-hdfs-sink.hdfs.closeTries
# hdfs sink尝试关闭文件的时间间隔，如果设置为0，表示不尝试，相当于于将hdfs.closeTries设置成1。默认值：180（秒）
# agentX.sinks.flume-hdfs-sink.hdfs.retryInterval
# 序列化类型。其他还有：avro_event或者是实现了EventSerializer.Builder的类名。默认值：TEXT
# agentX.sinks.flume-hdfs-sink.hdfs.serializer
```

注意：hdfs.rollInterval，hdfs.rollSize，hdfs.rollCount这3个参数尤为重要，因为这三个参数是控制HDFS文件滚动的，如果想要按照自己的方式做HDFS文件滚动必须三个参数都需要设置，我这里是按照300个Event来做HDFS文件滚动的，如果仅仅设置hdfs.rollCount一个参数是不起作用的，因为其他两个参数按照默认值还是会生效，如果只希望其中某些参数起作用，最好禁用其他的参数。

#### 在HDFS中查看

```
$ hdfs dfs -ls /flume/events/
16/09/23 14:43:04 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 1 items
drwxr-xr-x   - yunyu supergroup          0 2016-09-23 14:42 /flume/events/20160923

$ hdfs dfs -ls /flume/events/20160923/
16/09/23 14:43:15 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 4 items
-rw-r--r--   1 yunyu supergroup      92900 2016-09-23 14:42 /flume/events/20160923/events-.1474612925442
-rw-r--r--   1 yunyu supergroup       5880 2016-09-23 14:42 /flume/events/20160923/events-.1474612925443.tmp
-rw-r--r--   1 yunyu supergroup      92900 2016-09-23 14:42 /flume/events/20160923/events-.1474612930367
-rw-r--r--   1 yunyu supergroup      19193 2016-09-23 14:42 /flume/events/20160923/events-.1474612930368.tmp

# 使用hostname作为前缀，这里的127.0.0.1应该是从/etc/hosts配置文件中读取的
$ hdfs dfs -ls /flume/events/20160923
16/09/23 18:01:10 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 4 items
-rw-r--r--   1 yunyu supergroup      92900 2016-09-23 18:00 /flume/events/20160923/events-127.0.0.1-.1474624778493
-rw-r--r--   1 yunyu supergroup      25083 2016-09-23 18:00 /flume/events/20160923/events-127.0.0.1-.1474624778494.tmp
-rw-r--r--   1 yunyu supergroup      92900 2016-09-23 18:00 /flume/events/20160923/events-127.0.0.1-.1474624788628
-rw-r--r--   1 yunyu supergroup       5881 2016-09-23 18:00 /flume/events/20160923/events-127.0.0.1-.1474624788629.tmp
```

#### 遇到的问题和解决方法

```
2016-09-23 14:40:16,810 (SinkRunner-PollingRunner-DefaultSinkProcessor) [ERROR - org.apache.flume.SinkRunner$PollingRunner.run(SinkRunner.java:160)] Unable to deliver event. Exception follows.
org.apache.flume.EventDeliveryException: java.lang.NullPointerException: Expected timestamp in the Flume event headers, but it was null
	at org.apache.flume.sink.hdfs.HDFSEventSink.process(HDFSEventSink.java:463)
	at org.apache.flume.sink.DefaultSinkProcessor.process(DefaultSinkProcessor.java:68)
	at org.apache.flume.SinkRunner$PollingRunner.run(SinkRunner.java:147)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.NullPointerException: Expected timestamp in the Flume event headers, but it was null
	at com.google.common.base.Preconditions.checkNotNull(Preconditions.java:226)
	at org.apache.flume.formatter.output.BucketPath.replaceShorthand(BucketPath.java:228)
	at org.apache.flume.formatter.output.BucketPath.escapeString(BucketPath.java:432)
	at org.apache.flume.sink.hdfs.HDFSEventSink.process(HDFSEventSink.java:380)
	... 3 more
```

遇到上面的问题是因为写入到HDFS时，使用到了时间戳来区分目录结构，Flume的消息组件Event在接受到之后在Header中没有发现时间戳参数，导致该错误发生，有三种方法可以解决这个错误；

- 在Source中设置拦截器，为每条Event头中加入时间戳（效率会慢一些）

```
agentX.sources.flume-avro-sink.interceptors = i1
agentX.sources.flume-avro-sink.interceptors.i1.type = timestamp
```

- 设置使用本地的时间戳（如果客户端和flume集群时间不一致数据时间会不准确）

```
# 为sink指定该参数为true
agentX.sinks.flume-hdfs-sink.hdfs.useLocalTimeStamp = true 
```

- 在数据源头解决，在日志Event的Head中添加时间戳再再送到Flume（推荐使用）

在向Source发送Event时，将时间戳参数添加到Event的Header中即可，Header是一个Map，添加时MapKey为timestamp

参考文章：

- http://flume.apache.org/FlumeUserGuide.html#hdfs-sink
- http://lxw1234.com/archives/2015/10/527.htm
- http://blog.sina.com.cn/s/blog_db77b3c60102vrzt.html