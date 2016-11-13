---
title: "Hadoop学习（四）MapReduce清洗数据实例"
date: 2016-09-10 10:08:16
tags: [Hadoop原理架构体系, HDFS, MapReduce]
categories: [Hadoop]
---

通过前两篇的文章内容我们已经介绍了MapReduce的运行原理，以及WordCount实例的执行过程，接下来我们将根据我们的实际应用改写出一个清洗Log数据的MapReduce。

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

这里我们使用Maven来打包构建项目，同之前的WordCount实例是一个项目。我们也是将依赖的jar包也打包到birdHadoop.jar中，并且直接在pom文件中指定调用的入口类，注意这里我们修改了入口类是com.birdben.mapreduce.adlog.AdLogMain，需要在pom文件中配置如下

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
                        <mainClass>com.birdben.mapreduce.adlog.AdLogMain</mainClass>
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
# 执行我们的Shell脚本
$ sh scripts/mapreduce/runAdLog.sh
```

#### runAdLog.sh脚本文件

```
#!/bin/bash
local_path=~/Downloads/birdHadoop
hdfs_input_path=/birdben/input
hdfs_output_path=/birdben/output
# 在HDFS上创建需要分析的文件存储目录，如果已经存在就先删除再重新创建，保证脚本的正常执行
echo "删除HDFS上的input目录$hdfs_input_path"
hdfs dfs -rm -r $hdfs_input_path
echo "创建HDFS上的input目录$hdfs_input_path"
hdfs dfs -mkdir -p $hdfs_input_path
# 需要将我们要分析的track.log日志文件上传到HDFS文件目录下
echo "将$local_path/inputfile/AdLog/track.log文件复制到HDFS的目录$hdfs_input_path"
hdfs dfs -put $local_path/inputfile/AdLog/track.log $hdfs_input_path
# 需要先删除HDFS上已存在的目录，否则hadoop执行jar的时候会报错
echo "删除HDFS的output目录$hdfs_output_path"
hdfs dfs -rm -r -f $hdfs_output_path
# 需要在Maven的pom.xml文件中指定jar的入口类
echo "开始执行birdHadoop.jar..."
hadoop jar $local_path/target/birdHadoop.jar $hdfs_input_path $hdfs_output_path
echo "结束执行birdHadoop.jar..."

if [ ! -d $local_path/outputfile/AdLog ]; then
	# 如果本地文件目录不存在，就自动创建
	echo "自动创建$local_path/outputfile/AdLog目录"
	mkdir -p $local_path/outputfile/AdLog
else
	# 如果本地文件已经存在，就删除
	echo "删除$local_path/outputfile/AdLog/*目录下的所有文件"
	rm -rf $local_path/outputfile/AdLog/*
fi
# 从HDFS目录中导出mapreduce的结果文件到本地文件系统
echo "导出HDFS目录$hdfs_output_path目录下的文件到本地$local_path/outputfile/AdLog/"
hdfs dfs -get $hdfs_output_path/* $local_path/outputfile/AdLog/
```

下面是执行过程中的输出

```
$ sh scripts/mapreduce/runAdLog.sh 删除HDFS上的input目录/birdben/input16/11/02 20:03:21 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.Deleted /birdben/input创建HDFS上的input目录/birdben/input将/home/yunyu/Downloads/birdHadoop/inputfile/AdLog/track.log文件复制到HDFS的目录/birdben/input删除HDFS的output目录/birdben/output16/11/02 20:03:28 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.Deleted /birdben/output开始执行birdHadoop.jar...birdben out AdLog start16/11/02 20:03:30 INFO adlog.AdLogMain: birdben logger AdLog start16/11/02 20:03:31 INFO client.RMProxy: Connecting to ResourceManager at hadoop1/10.10.1.49:803216/11/02 20:03:33 INFO input.FileInputFormat: Total input paths to process : 116/11/02 20:03:33 INFO mapreduce.JobSubmitter: number of splits:116/11/02 20:03:33 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1478138258749_000116/11/02 20:03:33 INFO impl.YarnClientImpl: Submitted application application_1478138258749_000116/11/02 20:03:33 INFO mapreduce.Job: The url to track the job: http://hadoop1:8088/proxy/application_1478138258749_0001/16/11/02 20:03:33 INFO mapreduce.Job: Running job: job_1478138258749_000116/11/02 20:03:41 INFO mapreduce.Job: Job job_1478138258749_0001 running in uber mode : false16/11/02 20:03:41 INFO mapreduce.Job:  map 0% reduce 0%16/11/02 20:03:48 INFO mapreduce.Job:  map 100% reduce 0%16/11/02 20:03:54 INFO mapreduce.Job:  map 100% reduce 100%16/11/02 20:03:54 INFO mapreduce.Job: Job job_1478138258749_0001 completed successfully16/11/02 20:03:54 INFO mapreduce.Job: Counters: 49	File System Counters		FILE: Number of bytes read=1545		FILE: Number of bytes written=233699		FILE: Number of read operations=0		FILE: Number of large read operations=0		FILE: Number of write operations=0		HDFS: Number of bytes read=2509		HDFS: Number of bytes written=1503		HDFS: Number of read operations=6		HDFS: Number of large read operations=0		HDFS: Number of write operations=2	Job Counters 		Launched map tasks=1		Launched reduce tasks=1		Data-local map tasks=1		Total time spent by all maps in occupied slots (ms)=4100		Total time spent by all reduces in occupied slots (ms)=3026		Total time spent by all map tasks (ms)=4100		Total time spent by all reduce tasks (ms)=3026		Total vcore-seconds taken by all map tasks=4100		Total vcore-seconds taken by all reduce tasks=3026		Total megabyte-seconds taken by all map tasks=4198400		Total megabyte-seconds taken by all reduce tasks=3098624	Map-Reduce Framework		Map input records=9		Map output records=9		Map output bytes=1512		Map output materialized bytes=1545		Input split bytes=103		Combine input records=9		Combine output records=9		Reduce input groups=1		Reduce shuffle bytes=1545		Reduce input records=9		Reduce output records=9		Spilled Records=18		Shuffled Maps =1		Failed Shuffles=0		Merged Map outputs=1		GC time elapsed (ms)=169		CPU time spent (ms)=1450		Physical memory (bytes) snapshot=336318464		Virtual memory (bytes) snapshot=1343729664		Total committed heap usage (bytes)=136450048	Shuffle Errors		BAD_ID=0		CONNECTION=0		IO_ERROR=0		WRONG_LENGTH=0		WRONG_MAP=0		WRONG_REDUCE=0	File Input Format Counters 		Bytes Read=2406	File Output Format Counters 		Bytes Written=1503结束执行birdHadoop.jar...删除/home/yunyu/Downloads/birdHadoop/outputfile/AdLog/*目录下的所有文件导出HDFS目录/birdben/output目录下的文件到本地/home/yunyu/Downloads/birdHadoop/outputfile/AdLog/16/11/02 20:03:57 WARN hdfs.DFSClient: DFSInputStream has been closed already16/11/02 20:03:57 WARN hdfs.DFSClient: DFSInputStream has been closed already
```

Shell脚本的最后我们将HDFS文件导出到本地系统文件，查看一下这个目录下的文件。

```
$ ll outputfile/AdLog/total 12drwxrwxr-x 2 yunyu yunyu 4096 Nov  3 11:03 ./drwxrwxr-x 4 yunyu yunyu 4096 Oct 26 19:46 ../-rw-r--r-- 1 yunyu yunyu 1503 Nov  3 11:03 part-r-00000-rw-r--r-- 1 yunyu yunyu    0 Nov  3 11:03 _SUCCESS
```

查看一下我们所期望的结果文件part-r-00000的内容

```
$ cat outputfile/AdLog/part-r-00000 {"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475915093792","timestamp":1475915080561,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475913845544","timestamp":1475913832349,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475912715001","timestamp":1475912701768,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475915090789","timestamp":1475915077585,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932936","server_timestamp":"1475914840498","timestamp":1475914827284,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932928","server_timestamp":"1475914829332","timestamp":1475914816133,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932934","server_timestamp":"1475915090579","timestamp":1475915077351,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932930","server_timestamp":"1475914840425","timestamp":1475114827206,"ua":"","uid":"0"}{"bid":"0","device_id":"","did":"0","duid":"0","hb_uid":"0","rpid":"65351516503932932","server_timestamp":"1475914829286","timestamp":1475114816071,"ua":"","uid":"0"}
```

可以看到最终的结果是我们之前所期望的，大功告成 ^_^
