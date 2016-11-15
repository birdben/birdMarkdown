---
title: "Storm学习（三）Storm的WordCount实例"
date: 2016-11-07 14:57:13
tags: [Storm]
categories: [Storm]
---

之前写的Hadoop系列文章中，我们使用MapReduce实现了一个WordCount实例（就是统计一个文件中每个单词出现的次数），这里使用Storm来实现同样的功能。

我这里有一个Hadoop例子的项目，之前MapReduce相关的实例也放在该项目下。

- http://github.com/birdben/birdHadoop

下面就是我们要统计的文件内容

##### input_WordCount

```
Hadoop Hive HBaseSpark Hive HadoopKafka HBase ES Logstash StormFlume Kafka Hadoop
```

##### output_WordCount

```ES	1Flume	1HBase	2Hadoop	3Hive	2Kafka	2Logstash	1Spark	1Storm	1
```

### WordCount实例程序

实例程序请参考GitHub上的源代码

- http://github.com/birdben/birdHadoop

这里我们使用Maven来打包构建项目，pom文件中需要添加Storm相关jar的引用

```
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-core</artifactId>
    <version>0.10.2</version>
    <scope>provided</scope>
</dependency>
```

![Storm](http://img.blog.csdn.net/20161110192542351?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里我们要是实现的逻辑，WordReaderSpout负责从inputfile中读取文件的内容，将读取完成的文件重命名，读取之后的内容按行发送给WordSpliterBolt，由WordSpliter负责将一行内容按照空格进行拆分，拆分之后将拆分好的单词发送给WordCounterBolt，WordCounterBolt负责按照单词统计次数。整个Word处理的过程是一个Topology（拓扑图，我的理解就是一个任务的执行过程图），WordCountMain负责提交Topology到Storm集群。

- WordCountMain：提交Topology到Storm集群
- WordReaderSpout（Spout）：负责读取文件内容
- WordSpliterBolt（Bolt）：负责拆分每行内容中的单词
- WordCounterBolt（Bolt）：负责统计单词次数

注意：这里读取完文件一定要进行重命名操作，否则Storm集群会一直循环读取（因为代码中我们是扫描inputfile目录下除去.bak的所有文件的），而且Storm集群模式是不会停止的，这是Storm流式计算和MapReduce离线任务的本质区别。

Storm有两种运行模式，下面分别介绍这两种模式

- 本地模式：实际上本地模式在JVM中模拟了一个Storm集群，用于开发和测试Topology。在本地模式下运行Topology类似于在集群上运行Topology。只需使用LocalCluster类就可以创建一个进程内的集群。可以直接在IDE就可以启动Storm本地集群，可以在代码中控制集群的停止。

- 集群模式：需要将代码打包成jar包，然后在Storm集群机器上运行"storm jar birdHadoop.jar com.birdben.storm.demo.WordCountMain inputpath outputpath"命令，这样该Topology会运行在不同的JVM或物理机器上，并且可以在Storm UI中监控到。使用集群模式时，不能在代码中控制集群，这和LocalCluster是不一样的。无法在代码中控制集群的停止

#### 本地模式

本地模式需要在pom文件中引入Storm相应的jar包，这里需要注意scope这里要设置成compile，或者把scope去掉。因为我们是直接通过IDE启动Storm本地集群的，所以需要Storm相关的jar包。

```
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-core</artifactId>
    <version>0.10.2</version>
    <scope>compile</scope>
</dependency>
```

注意：在Maven打包之前需要先修改pom文件，指定我们的入口类是"com.birdben.storm.demo.WordCountMain"

本地模式提交WordCount这个Topology，但是休眠10秒中之后我们将kill掉WordCount这个Topology，这样才能够触发WordCounter中的cleanup方法，将我们的统计结果输出到目标文件中，否则的话，cleanup方法始终不会被调用，目标文件也是不会有统计结果的

```
LocalCluster cluster = new LocalCluster();
cluster.submitTopology("WordCount", conf, builder.createTopology());
Utils.sleep(10000);
cluster.killTopology("WordCount");
cluster.shutdown();
```

具体代码请参考：

- http://github.com/birdben/birdHadoop

```
# 进入项目根目录下
$ cd /Users/yunyu/workspace_git/birdHadoop
# 编译打包
$ mvn clean package
# 执行java -jar运行我们打好的jar包，这里将相关操作写成了Shell脚本
$ sh scripts/storm/runWordCount_Local.sh
```

#### runWordCount_Local.sh脚本文件

```
#!/bin/bash
local_path=~/workspace_git/birdHadoop
local_inputfile_path=$local_path/inputfile/WordCount
local_outputfile_path=$local_path/outputfile/WordCount
if [ -f $local_inputfile_path/input_WordCount.bak ]; then
        # 如果本地bak文件存在，就重命名去掉bak
        echo "正在重命名$local_inputfile_path/input_WordCount.bak文件"
        mv $local_inputfile_path/input_WordCount.bak $local_inputfile_path/input_WordCount
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
java -jar $local_path/target/birdHadoop.jar $local_inputfile_path $local_outputfile_path
echo "结束执行birdHadoop.jar..."
```

下面是执行过程中的输出

```
$ sh scripts/storm/runWordCount_Local.sh
正在重命名/Users/yunyu/workspace_git/birdHadoop/inputfile/WordCount/input_WordCount.bak文件
删除/Users/yunyu/workspace_git/birdHadoop/outputfile/WordCount/*目录下的所有文件
开始执行birdHadoop.jar...
log4j:WARN No appenders could be found for logger (backtype.storm.utils.Utils).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
WordCounter prepare out start
WordCounter clean out start
WordCounter result : hadoop 3
WordCounter result : hive 2
WordCounter result : logstash 1
WordCounter result : hbase 2
WordCounter result : flume 1
WordCounter result : kafka 2
WordCounter result : storm 1
WordCounter result : spark 1
WordCounter result : es 1
WordCounter clean out end
结束执行birdHadoop.jar...
```

查看一下我们所期望的结果文件output_WordCount的内容

```
$ cat outputfile/WordCount/output_WordCounthadoop 3
hive 2
logstash 1
hbase 2
flume 1
kafka 2
storm 1
spark 1
es 1
```

#### 集群模式

集群模式需要修改pom文件中Storm相应的jar包的scope设置成provided，否则再次引用就会冲突报错。因为我们是直接将打好的jar包提交到Storm集群中运行的，所以不需要Storm相关的jar包。

```
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-core</artifactId>
    <version>0.10.2</version>
    <scope>provided</scope>
</dependency>
```

如果没有将scope设置为provided，就会遇到如下错误

```
Caused by: java.io.IOException: Found multiple defaults.yaml resources. You're probably bundling the Storm jars with your topology jar. [jar:file:/data/storm-0.10.2/lib/storm-core-0.10.2.jar!/defaults.yaml, jar:file:/home/yunyu/Downloads/birdHadoop/target/birdHadoop.jar!/defaults.yaml]	at backtype.storm.utils.Utils.getConfigFileInputStream(Utils.java:266)	at backtype.storm.utils.Utils.findAndReadConfigFile(Utils.java:220)	... 103 more
```

集群模式提交Topology，需要将代码打包成jar包，然后在Storm集群机器上运行"storm jar birdHadoop.jar com.birdben.storm.demo.WordCountMain inputpath outputpath"命令，这样该Topology会运行在不同的JVM或物理机器上，并且可以在Storm UI中监控到。使用集群模式时，不能在代码中控制集群，这和LocalCluster是不一样的。无法在代码中控制集群的停止

```
StormSubmitter.submitTopology("WordCount", conf, builder.createTopology());
```

具体代码请参考：

- http://github.com/birdben/birdHadoop

```
# 进入项目根目录下
$ cd /Users/yunyu/workspace_git/birdHadoop
# 编译打包
$ mvn clean package
# 将打好的jar包上传到storm集群，执行storm jar运行我们打好的jar包，这里将相关操作写成了Shell脚本
$ sh scripts/storm/runWordCount_Remote.sh
```

这里需要注意：

上一篇我们说过，在Storm的集群里面有两种节点： 控制节点(master node)和工作节点(worker node)。控制节点上面运行一个后台程序： Nimbus， 它的作用类似Hadoop里面的JobTracker。Nimbus负责在集群里面分布代码，分配工作给机器， 并且监控状态。

每一个工作节点上面运行一个叫做Supervisor的节点（类似 TaskTracker）。Supervisor会监听分配给它那台机器的工作，根据需要 启动/关闭工作进程。每一个工作进程执行一个Topology（类似 Job）的一个子集；一个运行的Topology由运行在很多机器上的很多工作进程 Worker（类似 Child）组成。

在我们的WordCount实例中，storm集群是三台机器，hadoop1是Nimbus，hadoop2和hadoop3是Supervisor，这里执行storm jar（也就是提交Topology）无论是在哪台机器上操作，最后都应该是在Supervisor分配的Work进程中进行计算的，所以我们inputfile文件应该上传到所有Storm集群的机器上，这样才能够避免Work进程在读取inputfile文件读取不到。这个问题我也是犯浑了很久，之前一直在Nimbus机器执行shell脚本发现WordReader读取文件的时候一直读取不到，我一直以为在Nimbus机器上执行storm jar（提交Topology）就一定是在Nimbus机器上执行操作，后来仔细观察了日志才发现WordCount的运行日志根本不在Nimbus机器上，而是在Supervisor机器上，所以才知道原来运行WordReader的Work进程在Supervisor机器上，看来对于Storm的运行原理还是理解的不够深。

#### runWordCount_Remote.sh脚本文件

```
#!/bin/bashlocal_path=~/Downloads/birdHadooplocal_inputfile_path=$local_path/inputfile/WordCountlocal_outputfile_path=$local_path/outputfile/WordCount
main_class=com.birdben.storm.demo.WordCountMainif [ -f $local_inputfile_path/input_WordCount.bak ]; then        # 如果本地bak文件存在，就重命名去掉bak        echo "正在重命名$local_inputfile_path/input_WordCount.bak文件"        mv $local_inputfile_path/input_WordCount.bak $local_inputfile_path/input_WordCountfiif [ ! -d $local_outputfile_path ]; then        # 如果本地文件目录不存在，就自动创建        echo "自动创建$outputfile_path目录"        mkdir -p $local_outputfile_pathelse        # 如果本地文件已经存在，就删除        echo "删除$local_outputfile_path/*目录下的所有文件"        rm -rf $local_outputfile_path/*fi# 需要在Maven的pom.xml文件中指定jar的入口类echo "开始执行birdHadoop.jar..."storm jar $local_path/target/birdHadoop.jar $main_class $local_inputfile_path $local_outputfile_pathecho "结束执行birdHadoop.jar..."
```

执行Shell脚本之后，可以在Storm UI中查看到Topology Summary中多了一个WordCounter Topology，Topology Id是WordCount-6-1479006604，我们找到Supervisor机器上的log日志（${STORM_HOME}/logs），该日志目录下会根据Topology Id生成对应的日志文件如下：

- WordCount-6-1479006604-worker-6703.log- WordCount-6-1479006604-worker-6703.log.err- WordCount-6-1479006604-worker-6703.log.metrics.log- WordCount-6-1479006604-worker-6703.log.out

我们可以查看一下WordCount-6-1479006604-worker-6703.log日志，我们代码中的日志输出都在这个日志文件中，可以看到Storm集群读取我们指定的inputfile，并且按照指定方式拆分出Word单词。

```
$ vi WordCount-6-1479006604-worker-6703.log
...
2016-11-12 19:10:11.001 b.s.d.executor [INFO] Preparing bolt word-spilter:(4)2016-11-12 19:10:11.002 b.s.d.executor [INFO] Prepared bolt word-spilter:(4)2016-11-12 19:10:11.002 b.s.d.executor [INFO] Processing received message FOR 4 TUPLE: source: word-reader:3, stream: default, id: {}, [Hadoop Hive HBase]2016-11-12 19:10:11.002 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.002 STDIO [INFO] WordSpliter execute out start2016-11-12 19:10:11.004 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.005 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.006 b.s.d.task [INFO] Emitting: word-spilter default [hadoop]2016-11-12 19:10:11.006 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.007 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [hadoop]2016-11-12 19:10:11.007 b.s.d.task [INFO] Emitting: word-spilter default [hive]2016-11-12 19:10:11.008 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [hive]2016-11-12 19:10:11.008 b.s.d.task [INFO] Emitting: word-spilter default [hbase]2016-11-12 19:10:11.008 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.008 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [hbase]2016-11-12 19:10:11.009 STDIO [INFO] WordSpliter execute out end2016-11-12 19:10:11.009 b.s.d.executor [INFO] BOLT ack TASK: 4 TIME:  TUPLE: source: word-reader:3, stream: default, id: {}, [Hadoop Hive HBase]2016-11-12 19:10:11.009 b.s.d.executor [INFO] Execute done TUPLE source: word-reader:3, stream: default, id: {}, [Hadoop Hive HBase] TASK: 4 DELTA:2016-11-12 19:10:11.010 b.s.d.executor [INFO] Processing received message FOR 4 TUPLE: source: word-reader:3, stream: default, id: {}, [Spark Hive Hadoop]2016-11-12 19:10:11.010 STDIO [INFO] WordSpliter execute out start2016-11-12 19:10:11.010 b.s.d.task [INFO] Emitting: word-spilter default [spark]2016-11-12 19:10:11.010 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.010 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [spark]2016-11-12 19:10:11.011 b.s.d.task [INFO] Emitting: word-spilter default [hive]2016-11-12 19:10:11.011 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [hive]2016-11-12 19:10:11.011 b.s.d.task [INFO] Emitting: word-spilter default [hadoop]2016-11-12 19:10:11.012 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [hadoop]2016-11-12 19:10:11.012 STDIO [INFO] WordSpliter execute out end2016-11-12 19:10:11.012 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount2016-11-12 19:10:11.012 b.s.d.executor [INFO] BOLT ack TASK: 4 TIME:  TUPLE: source: word-reader:3, stream: default, id: {}, [Spark Hive Hadoop]2016-11-12 19:10:11.012 b.s.d.executor [INFO] Execute done TUPLE source: word-reader:3, stream: default, id: {}, [Spark Hive Hadoop] TASK: 4 DELTA:2016-11-12 19:10:11.013 b.s.d.executor [INFO] Processing received message FOR 4 TUPLE: source: word-reader:3, stream: default, id: {}, [Kafka HBase ES Logstash Storm]2016-11-12 19:10:11.013 STDIO [INFO] WordSpliter execute out start2016-11-12 19:10:11.013 b.s.d.task [INFO] Emitting: word-spilter default [kafka]2016-11-12 19:10:11.013 b.s.d.executor [INFO] TRANSFERING tuple TASK: 2 TUPLE: source: word-spilter:4, stream: default, id: {}, [kafka]2016-11-12 19:10:11.014 b.s.d.task [INFO] Emitting: word-spilter default [hbase]2016-11-12 19:10:11.014 STDIO [INFO] out inputPath:/home/yunyu/Downloads/birdHadoop/inputfile/WordCount
...
```

上述步骤都已经执行完毕了，日志也没有错误了，但是查看outputfile文件还是没有内容。这是因为我们的WordCount的cleanup方法没有被执行，所以并没有将我们的统计结果输出到outputfile文件中。这里我们是集群模式，因为Storm是流式计算引擎，所以集群的WordCount Topology不停止是不会调用cleanup方法的。所以这里我们需要使用storm kill WordCount方式杀掉WordCount Topology，这样才能够使Storm调用WordCount的cleanup方法将统计结果输出到outputfile中。

```
$ storm kill WordCountRunning: /usr/local/java/bin/java -client -Ddaemon.name= -Dstorm.options= -Dstorm.home=/data/storm-0.10.2 -Dstorm.log.dir=/data/storm-0.10.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /data/storm-0.10.2/lib/kryo-2.21.jar:/data/storm-0.10.2/lib/servlet-api-2.5.jar:/data/storm-0.10.2/lib/hadoop-auth-2.4.0.jar:/data/storm-0.10.2/lib/minlog-1.2.jar:/data/storm-0.10.2/lib/storm-core-0.10.2.jar:/data/storm-0.10.2/lib/log4j-core-2.1.jar:/data/storm-0.10.2/lib/reflectasm-1.07-shaded.jar:/data/storm-0.10.2/lib/clojure-1.6.0.jar:/data/storm-0.10.2/lib/disruptor-2.10.4.jar:/data/storm-0.10.2/lib/log4j-over-slf4j-1.6.6.jar:/data/storm-0.10.2/lib/asm-4.0.jar:/data/storm-0.10.2/lib/log4j-slf4j-impl-2.1.jar:/data/storm-0.10.2/lib/slf4j-api-1.7.7.jar:/data/storm-0.10.2/lib/log4j-api-2.1.jar:/usr/local/storm/conf:/data/storm-0.10.2/bin backtype.storm.command.kill_topology WordCount1467 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources1535 [main] INFO  b.s.u.Utils - Using storm.yaml from resources2180 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources2200 [main] INFO  b.s.u.Utils - Using storm.yaml from resources2227 [main] INFO  b.s.thrift - Connecting to Nimbus at hadoop1:6627 as user: 2228 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources2251 [main] INFO  b.s.u.Utils - Using storm.yaml from resources2269 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [2000] the maxSleepTimeMs [60000] the maxRetries [5]
```

这里还有个需要注意的地方，就是storm kill WordCount不是立即执行完毕的，它只是将WordCount的Topology状态先标记成KILLED，还需要sleep一段时间之后Topology才会真正被Kill。所以执行完storm kill之后在Storm UI中仍然能查看到WordCount的Topology，只是状态变成KILLED。如果此时再次执行Shell脚本重新运行WordCount Topology，Storm集群仍然会提示WordCount Topology已经存在了。


参考文章：

- http://storm.apache.org/releases/1.0.2/Setting-up-a-Storm-cluster.html
- http://book.51cto.com/art/201410/453401.htm
- http://blog.csdn.net/guoqiangma/article/details/38045259
- http://blog.csdn.net/xeseo/article/details/18219183
- http://blog.itpub.net/28912557/viewspace-1450885/