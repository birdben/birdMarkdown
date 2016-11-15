---
title: "Storm学习（四）Storm清洗数据实例"
date: 2016-11-15 16:47:32
tags: [Storm]
categories: [Storm]
---

之前学习Hadoop的时候，使用MapReduce做了一个track.log日志文件的数据清洗实例，按照我们的需要提取出有用的日志数据，这里我们使用Storm来实现同样的功能。

具体源代码请关注下面的GitHub项目

- http://github.com/birdben/birdHadoop

### 数据清洗的目标

这里我们期望将下面的track.log日志文件内容转化一下，将logs外层结构去掉，提起出来logs的内层数据，并且将原来的logs下的数组转换成多条新的日志记录。

##### track.log日志文件

```
{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}
{"logs":[{"timestamp":"1475114827206","rpid":"65351516503932930","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840425}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.425Z"}
{"logs":[{"timestamp":"1475915077351","rpid":"65351516503932934","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090579}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.579Z"}
{"logs":[{"timestamp":"1475914816133","rpid":"65351516503932928","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829332}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.332Z"}
{"logs":[{"timestamp":"1475914827284","rpid":"65351516503932936","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840498}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.499Z"}
{"logs":[{"timestamp":"1475915077585","rpid":"65351516503932932","name":"birdben.ad.view_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915090789}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:50.789Z"}
{"logs":[{"timestamp":"1475912701768","rpid":"65351516503932930","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475912715001}],"level":"info","message":"logs","timestamp":"2016-10-08T07:45:15.001Z"}
{"logs":[{"timestamp":"1475913832349","rpid":"65351516503932934","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475913845544}],"level":"info","message":"logs","timestamp":"2016-10-08T08:04:05.544Z"}
{"logs":[{"timestamp":"1475915080561","rpid":"65351516503932928","name":"birdben.ad.click_ad","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475915093792}],"level":"info","message":"logs","timestamp":"2016-10-08T08:24:53.792Z"}
```

##### 期望清洗之后的文件内容如下

```
{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475915093792","timestamp":1475915080561,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475913845544","timestamp":1475913832349,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475912715001","timestamp":1475912701768,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475915090789","timestamp":1475915077585,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932936","server_timestamp":"1475914840498","timestamp":1475914827284,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475914829332","timestamp":1475914816133,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475915090579","timestamp":1475915077351,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475914840425","timestamp":1475114827206,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}
```

### AdLog实例程序

实例程序请参考GitHub上的源代码

- http://github.com/birdben/birdHadoop

这里我们使用Maven来打包构建项目，同之前的MapReduce相关实例是一个项目，我们用了分开在不同的package中。

```
# 进入项目根目录下
$ cd /Users/yunyu/workspace_git/birdHadoop
# 编译打包
$ mvn clean package
# 执行我们的Shell脚本
$ sh scripts/storm/runAdLog.sh
```

#### runAdLog.sh脚本文件

```
#!/bin/bash
local_path=~/Downloads/birdHadoop
local_inputfile_path=$local_path/inputfile/AdLog
local_outputfile_path=$local_path/outputfile/AdLog
main_class=com.birdben.storm.adlog.AdLogMain
if [ -f $local_inputfile_path/track.log.bak ]; then
	# 如果本地bak文件存在，就重命名去掉bak
	echo "正在重命名$local_inputfile_path/track.log.bak文件"
	mv $local_inputfile_path/track.log.bak $local_inputfile_path/track.log
fi
if [ ! -d $local_outputfile_path ]; then
	# 如果本地文件目录不存在，就自动创建
	echo "自动创建$outputfile_path目录"
	mkdir -p $local_outputfile_path
else
	# 如果本地文件已经存在，就删除
	echo "删除$local_outputfile_path/*目录下的所有文件"
	rm -rf $local_outputfile_path/*
fi
# 需要在Maven的pom.xml文件中指定jar的入口类
echo "开始执行birdHadoop.jar..."
storm jar $local_path/target/birdHadoop.jar $main_class $local_inputfile_path $local_outputfile_path
echo "结束执行birdHadoop.jar..."
```

注意：这里使用的集群模式运行的，inputfile文件需要上传到Storm的Supervisor机器上，否则Storm运行的时候会找不到inputfile文件。

执行Shell脚本之后，可以在Storm UI中查看到Topology Summary中多了一个AdLog Topology，Topology Id是AdLog-1-1479198597，我们找到Supervisor机器上的log日志（${STORM_HOME}/logs），该日志目录下会根据Topology Id生成对应的日志文件如下：

- AdLog-1-1479198597-worker-6703.log- AdLog-1-1479198597-worker-6703.log.err- AdLog-1-1479198597-worker-6703.log.metrics.log- AdLog-1-1479198597-worker-6703.log.out

我们可以查看一下AdLog-1-1479198597-worker-6703.log日志，我们代码中的日志输出都在这个日志文件中，可以看到Storm集群读取我们指定的inputfile，并且按照指定方式提取我们需要的日志。

```
$ vi AdLog-1-1479198597-worker-6703.log
...2016-11-15 00:30:04.316 b.s.d.executor [INFO] Preparing bolt adlog-counter:(2)2016-11-15 00:30:04.318 STDIO [INFO] AdLogCounterBolt prepare out start2016-11-15 00:30:04.319 b.s.d.executor [INFO] Prepared bolt adlog-counter:(2)2016-11-15 00:30:04.338 b.s.d.executor [INFO] Preparing bolt adlog-parser:(3)2016-11-15 00:30:04.340 b.s.d.executor [INFO] Prepared bolt adlog-parser:(3)2016-11-15 00:30:04.340 b.s.d.executor [INFO] Processing received message FOR 3 TUPLE: source: adlog-reader:4, stream: default, id: {}, [{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}]2016-11-15 00:30:04.341 STDIO [INFO] AdLogParserBolt execute out start2016-11-15 00:30:04.356 STDIO [ERROR] SLF4J: Detected both log4j-over-slf4j.jar AND slf4j-log4j12.jar on the class path, preempting StackOverflowError.2016-11-15 00:30:04.356 STDIO [ERROR] SLF4J: See also http://www.slf4j.org/codes.html#log4jDelegationLoop for more details.2016-11-15 00:30:04.361 STDIO [INFO] birdben AdLogParser out start2016-11-15 00:30:04.367 STDIO [ERROR] Nov 15, 2016 12:30:04 AM com.birdben.mapreduce.adlog.parser.AdLogParser convertLogToAdINFO: birdben AdLogParser logger start2016-11-15 00:30:04.435 STDIO [ERROR] Nov 15, 2016 12:30:04 AM com.birdben.mapreduce.adlog.parser.AdLogParser convertLogToAdINFO: convertLogToAd name:birdben.ad.open_hb2016-11-15 00:30:04.452 b.s.d.task [INFO] Emitting: adlog-parser default [{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}]2016-11-15 00:30:04.453 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: adlog-parser:3, stream: default, id: {}, [{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}]2016-11-15 00:30:04.454 STDIO [INFO] AdLogParserBolt execute out end2016-11-15 00:30:04.454 b.s.d.executor [INFO] Processing received message FOR 2 TUPLE: source: adlog-parser:3, stream: default, id: {}, [{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}]2016-11-15 00:30:04.454 STDIO [INFO] AdLogCounterBolt execute out start2016-11-15 00:30:04.454 b.s.d.executor [INFO] BOLT ack TASK: 3 TIME:  TUPLE: source: adlog-reader:4, stream: default, id: {}, [{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}]2016-11-15 00:30:04.454 STDIO [INFO] AdLogCounterBolt execute out end2016-11-15 00:30:04.455 b.s.d.executor [INFO] Execute done TUPLE source: adlog-reader:4, stream: default, id: {}, [{"logs":[{"timestamp":"1475114816071","rpid":"65351516503932932","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914829286}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:29.286Z"}] TASK: 3 DELTA:2016-11-15 00:30:04.455 b.s.d.executor [INFO] BOLT ack TASK: 2 TIME: 1 TUPLE: source: adlog-parser:3, stream: default, id: {}, [{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}]2016-11-15 00:30:04.455 b.s.d.executor [INFO] Processing received message FOR 3 TUPLE: source: adlog-reader:4, stream: default, id: {}, [{"logs":[{"timestamp":"1475114827206","rpid":"65351516503932930","name":"birdben.ad.open_hb","bid":0,"uid":0,"did":0,"duid":0,"hb_uid":0,"ua":"","device_id":"","server_timestamp":1475914840425}],"level":"info","message":"logs","timestamp":"2016-10-08T08:20:40.425Z"}]2016-11-15 00:30:04.455 STDIO [INFO] AdLogParserBolt execute out start2016-11-15 00:30:04.455 STDIO [INFO] birdben AdLogParser out start
...
```

Storm数据清洗运行成功后，需要像之前一样kill掉AdLog Topology之后才会调用cleanup方法将清洗后的日志输出到outputfile文件中

```
$ storm kill AdLogRunning: /usr/local/java/bin/java -client -Ddaemon.name= -Dstorm.options= -Dstorm.home=/data/storm-0.10.2 -Dstorm.log.dir=/data/storm-0.10.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /data/storm-0.10.2/lib/storm-core-0.10.2.jar:/data/storm-0.10.2/lib/slf4j-api-1.7.7.jar:/data/storm-0.10.2/lib/clojure-1.6.0.jar:/data/storm-0.10.2/lib/disruptor-2.10.4.jar:/data/storm-0.10.2/lib/servlet-api-2.5.jar:/data/storm-0.10.2/lib/log4j-api-2.1.jar:/data/storm-0.10.2/lib/log4j-core-2.1.jar:/data/storm-0.10.2/lib/minlog-1.2.jar:/data/storm-0.10.2/lib/reflectasm-1.07-shaded.jar:/data/storm-0.10.2/lib/log4j-over-slf4j-1.6.6.jar:/data/storm-0.10.2/lib/asm-4.0.jar:/data/storm-0.10.2/lib/hadoop-auth-2.4.0.jar:/data/storm-0.10.2/lib/kryo-2.21.jar:/data/storm-0.10.2/lib/log4j-slf4j-impl-2.1.jar:/usr/local/storm/conf:/data/storm-0.10.2/bin backtype.storm.command.kill_topology AdLog1331 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources1401 [main] INFO  b.s.u.Utils - Using storm.yaml from resources1954 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources1971 [main] INFO  b.s.u.Utils - Using storm.yaml from resources1987 [main] INFO  b.s.thrift - Connecting to Nimbus at hadoop1:6627 as user: 1987 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources2024 [main] INFO  b.s.u.Utils - Using storm.yaml from resources2045 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [2000] the maxSleepTimeMs [60000] the maxRetries [5]2094 [main] INFO  b.s.c.kill-topology - Killed topology: AdLog
```

查看一下我们所期望的结果文件的内容

```
$ cat outputfile/AdLog/output_AdLog {"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475914840425","timestamp":1475114827206,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475915090579","timestamp":1475915077351,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475914829332","timestamp":1475914816133,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932936","server_timestamp":"1475914840498","timestamp":1475914827284,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475915090789","timestamp":1475915077585,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475912715001","timestamp":1475912701768,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475913845544","timestamp":1475913832349,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475915093792","timestamp":1475915080561,"ua":"","uid":"0"}
```

