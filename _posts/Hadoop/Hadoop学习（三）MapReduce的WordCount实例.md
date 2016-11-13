---
title: "Hadoop学习（三）MapReduce的WordCount实例"
date: 2016-09-10 10:08:16
tags: [Hadoop原理架构体系, HDFS, MapReduce]
categories: [Hadoop]
---

前面Hadoop的集群环境我们已经搭建好了，而且也分析了MapReduce和YARN之前的关系，以及在Hadoop中的作用，接下来我们将在Hadoop集群环境中跑一个我们自己创建的MapReduce离线任务实例WordCount。

我这里有一个Hadoop例子的项目，是我自己写的一些大数据相关的实例，后续会持续更新的。

- http://github.com/birdben/birdHadoop

WordCount的实例很简单，就是要统计一下某一个文件中每次单词出现的次数，下面就是我们要统计的文件内容

##### input_WordCount

```
Hadoop Hive HBaseSpark Hive HadoopKafka HBase ES Logstash StormFlume Kafka Hadoop
```

##### output_WordCount

```ES	1Flume	1HBase	2Hadoop	3Hive	2Kafka	2Logstash	1Spark	1Storm	1
```

下面引用了其他博客的图，因为这些图十分形象的描述了MapReduce的执行过程，博客原文链接如下：

- http://www.cnblogs.com/xia520pi/archive/2012/05/16/2504205.html

![WordCount1](http://img.blog.csdn.net/20161030182309567?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 将文件拆分成splits，由于测试用的文件较小，所以每个文件为一个split，并将文件按行分割形成<key,value>对，如上图所示。这一步由MapReduce框架自动完成，其中偏移量（即key值）包括了回车所占的字符数（Windows和Linux环境会不同）。

![WordCount2](http://img.blog.csdn.net/20161030182342598?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 将分割好的<key,value>对交给用户定义的map方法进行处理，生成新的<key,value>对，如上图所示。

![WordCount3](http://img.blog.csdn.net/20161030182404755?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 得到map方法输出的<key,value>对后，Mapper会将它们按照key值进行排序，并执行Combine过程，将key至相同value值累加，得到Mapper的最终输出结果。如上图所示。

![WordCount4](http://img.blog.csdn.net/20161030182418630?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- Reducer先对从Mapper接收的数据进行排序，再交由用户自定义的reduce方法进行处理，得到新的<key,value>对，并作为WordCount的输出结果，如上图所示。

### WordCount实例程序

实例程序请参考GitHub上的源代码

- http://github.com/birdben/birdHadoop

这里我们使用Maven来打包构建项目，pom文件中需要添加Hadoop相关jar的引用

```
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>2.4.0</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-mapreduce-client-core</artifactId>
    <version>2.4.0</version>
</dependency>
```

这里Maven构建将依赖的jar包也打包到birdHadoop.jar中，并且直接在pom文件中指定调用的入口类，我这里指定了入口类是com.birdben.mapreduce.demo.WordCount，然后运行java -jar birdHadoop.jar inputfile outputfile即可。pom文件中的配置如下

```
<build>
    <finalName>birdHadoop</finalName>
    <plugins>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <appendAssemblyId>false</appendAssemblyId>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifest>
                        <mainClass>com.birdben.mapreduce.demo.WordCount</mainClass>
                    </manifest>
                </archive>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>assembly</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.3</version>
            <configuration>
                <source>1.7</source>
                <target>1.7</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```
# 进入项目根目录下
$ cd /Users/yunyu/workspace_git/birdHadoop
# 编译打包
$ mvn clean package
# 执行我们的Shell脚本，这里将HDFS的相关操作写成了Shell脚本
$ sh scripts/mapreduce/runWordCount.sh
```

#### runWordCount.sh脚本文件

```
#!/bin/bashlocal_path=~/Downloads/birdHadoophdfs_input_path=/birdben/inputhdfs_output_path=/birdben/output# 在HDFS上创建需要分析的文件存储目录，如果已经存在就先删除再重新创建，保证脚本的正常执行echo "删除HDFS上的input目录$hdfs_input_path"hdfs dfs -rm -r $hdfs_input_pathecho "创建HDFS上的input目录$hdfs_input_path"hdfs dfs -mkdir -p $hdfs_input_path# 需要将我们要分析的track.log日志文件上传到HDFS文件目录下echo "将$local_path/inputfile/WordCount/input_WordCount文件复制到HDFS的目录$hdfs_input_path"hdfs dfs -put $local_path/inputfile/WordCount/input_WordCount $hdfs_input_path# 需要先删除HDFS上已存在的目录，否则hadoop执行jar的时候会报错echo "删除HDFS的output目录$hdfs_output_path"hdfs dfs -rm -r -f $hdfs_output_path# 需要在Maven的pom.xml文件中指定jar的入口类echo "开始执行birdHadoop.jar..."hadoop jar $local_path/target/birdHadoop.jar $hdfs_input_path $hdfs_output_pathecho "结束执行birdHadoop.jar..."if [ ! -d $local_path/outputfile/WordCount ]; then	# 如果本地文件目录不存在，就自动创建	echo "自动创建$local_path/outputfile/WordCount目录"	mkdir -p $local_path/outputfile/WordCountelse	# 如果本地文件已经存在，就删除	echo "删除$local_path/outputfile/WordCount/*目录下的所有文件"	rm -rf $local_path/outputfile/WordCount/*fi# 从HDFS目录中导出mapreduce的结果文件到本地文件系统echo "导出HDFS目录$hdfs_output_path目录下的文件到本地$local_path/outputfile/WordCount/"hdfs dfs -get $hdfs_output_path/* $local_path/outputfile/WordCount/
```

下面是执行过程中的输出

```
$ sh scripts/mapreduce/runWordCount.sh
删除HDFS上的input目录/birdben/input16/11/02 05:12:57 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.Deleted /birdben/input创建HDFS上的input目录/birdben/input将/home/yunyu/Downloads/birdHadoop/inputfile/WordCount/input_WordCount文件复制到HDFS的目录/birdben/input删除HDFS的output目录/birdben/output16/11/02 05:13:04 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.Deleted /birdben/output开始执行birdHadoop.jar...birdben out WordCount start16/11/02 05:13:07 INFO demo.WordCount: birdben logger WordCount start16/11/02 05:13:08 INFO client.RMProxy: Connecting to ResourceManager at hadoop1/10.10.1.49:803216/11/02 05:13:09 INFO input.FileInputFormat: Total input paths to process : 116/11/02 05:13:09 INFO mapreduce.JobSubmitter: number of splits:116/11/02 05:13:10 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1478088725123_000116/11/02 05:13:10 INFO impl.YarnClientImpl: Submitted application application_1478088725123_000116/11/02 05:13:10 INFO mapreduce.Job: The url to track the job: http://hadoop1:8088/proxy/application_1478088725123_0001/16/11/02 05:13:10 INFO mapreduce.Job: Running job: job_1478088725123_000116/11/02 05:13:17 INFO mapreduce.Job: Job job_1478088725123_0001 running in uber mode : false16/11/02 05:13:17 INFO mapreduce.Job:  map 0% reduce 0%16/11/02 05:13:24 INFO mapreduce.Job:  map 100% reduce 0%16/11/02 05:13:32 INFO mapreduce.Job:  map 100% reduce 100%16/11/02 05:13:32 INFO mapreduce.Job: Job job_1478088725123_0001 completed successfully16/11/02 05:13:32 INFO mapreduce.Job: Counters: 49	File System Counters		FILE: Number of bytes read=114		FILE: Number of bytes written=230785		FILE: Number of read operations=0		FILE: Number of large read operations=0		FILE: Number of write operations=0		HDFS: Number of bytes read=194		HDFS: Number of bytes written=72		HDFS: Number of read operations=6		HDFS: Number of large read operations=0		HDFS: Number of write operations=2	Job Counters 		Launched map tasks=1		Launched reduce tasks=1		Data-local map tasks=1		Total time spent by all maps in occupied slots (ms)=4836		Total time spent by all reduces in occupied slots (ms)=3355		Total time spent by all map tasks (ms)=4836		Total time spent by all reduce tasks (ms)=3355		Total vcore-seconds taken by all map tasks=4836		Total vcore-seconds taken by all reduce tasks=3355		Total megabyte-seconds taken by all map tasks=4952064		Total megabyte-seconds taken by all reduce tasks=3435520	Map-Reduce Framework		Map input records=4		Map output records=14		Map output bytes=141		Map output materialized bytes=114		Input split bytes=109		Combine input records=14		Combine output records=9		Reduce input groups=9		Reduce shuffle bytes=114		Reduce input records=9		Reduce output records=9		Spilled Records=18		Shuffled Maps =1		Failed Shuffles=0		Merged Map outputs=1		GC time elapsed (ms)=54		CPU time spent (ms)=1330		Physical memory (bytes) snapshot=455933952		Virtual memory (bytes) snapshot=1415868416		Total committed heap usage (bytes)=276299776	Shuffle Errors		BAD_ID=0		CONNECTION=0		IO_ERROR=0		WRONG_LENGTH=0		WRONG_MAP=0		WRONG_REDUCE=0	File Input Format Counters 		Bytes Read=85	File Output Format Counters 		Bytes Written=72结束执行birdHadoop.jar...删除/home/yunyu/Downloads/birdHadoop/outputfile/WordCount/*目录下的所有文件导出HDFS目录/birdben/output目录下的文件到本地/home/yunyu/Downloads/birdHadoop/outputfile/WordCount/16/11/02 05:13:34 WARN hdfs.DFSClient: DFSInputStream has been closed already16/11/02 05:13:35 WARN hdfs.DFSClient: DFSInputStream has been closed already
```

Shell脚本的最后我们将HDFS文件导出到本地系统文件，查看一下这个目录下的文件。

```
$ ll outputfile/WordCount/total 12drwxrwxr-x 2 yunyu yunyu 4096 Nov  2 20:13 ./drwxrwxr-x 4 yunyu yunyu 4096 Oct 26 19:46 ../-rw-r--r-- 1 yunyu yunyu   72 Nov  2 20:13 part-r-00000-rw-r--r-- 1 yunyu yunyu    0 Nov  2 20:13 _SUCCESS
```

查看一下我们所期望的结果文件part-r-00000的内容

```
$ cat outputfile/WordCount/part-r-00000 ES	1Flume	1HBase	2Hadoop	3Hive	2Kafka	2Logstash	1Spark	1Storm	1
```

参考文章：

- http://www.cnblogs.com/xwdreamer/archive/2011/01/04/2297049.html
- http://blog.csdn.net/lisonglisonglisong/article/details/47125319
- http://www.cnblogs.com/xia520pi/archive/2012/05/16/2504205.html