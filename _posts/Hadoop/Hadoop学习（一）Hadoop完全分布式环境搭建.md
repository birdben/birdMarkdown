---
title: "Hadoop学习（一）Hadoop完全分布式环境搭建"
date: 2016-09-10 16:09:30
tags: [Hadoop原理架构体系, HDFS]
categories: [Hadoop]
---

今天学习的信息量有点大收获不少，一时之间不知道从哪里开始写，希望尽量把我今天学习到的东西记录下来，因为内容太多可能会分几篇记录。其实之前有写过一篇用Docker搭建Hadoop环境的文章，当时其实搭建的是单机伪分布式的环境，今天这里搭建的是Hadoop完全分布式环境。今天又看了许多文章，对于Hadoop的体系架构又有了一定新的理解，包括1.x版本和2.x版本的不同。

### Hadoop集群环境

我这里使用的三台虚拟机，每台虚拟机有自己的独立IP

```
192.168.1.119   hadoop1192.168.1.150   hadoop2192.168.1.149   hadoop3
```
相关环境信息

```
操作系统: Ubuntu 14.04.5 LTS
JDK版本: 1.7.0_79
Hadoop版本: 2.7.1
```

### JDK安装

省略

### Hadoop安装
```
# 下载Hadoop安装包
$ curl -O http://mirrors.cnnic.cn/apache/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz

# 解压Hadoop压缩包
$ tar -zxvf hadoop-2.7.1.tar.gz
```

### Hadoop集群配置

注意：以下配置请根据自己的实际环境修改

##### 配置环境变量/etc/profile
```
JAVA_HOME=/usr/local/javaexport JAVA_HOMEHADOOP_HOME=/usr/local/hadoopexport HADOOP_HOMEPATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$PATHexport PATH
```

##### 配置HADOOP_HOME/etc/hadoop/hadoop-env.sh，添加以下内容
```
export JAVA_HOME=/usr/local/java
export PATH=$PATH:$HADOOP_HOME/bin
```

##### 配置HADOOP_HOME/etc/hadoop/yarn-env.sh，添加以下内容
```
export JAVA_HOME=/usr/local/java
```

##### 配置HADOOP_HOME/etc/hadoop/core-site.xml
这里我使用Hadoop1这台虚拟机作为NameNode节点

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://Hadoop1:9000</value>
  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/hdfs-site.xml
```
<configuration>
  <!-- 分布式文件系统数据块复制数，我们这里是Hadoop2和Hadoop3两个节点 -->  <property>    <name>dfs.replication</name>    <value>2</value>  </property>
  <!-- DFS namenode存放name table的目录 -->  <property>    <name>dfs.namenode.name.dir</name>    <value>file:/data/hdfs/name</value>  </property>
  <!-- DFS datanode存放数据block的目录 -->  <property>    <name>dfs.datanode.data.dir</name>    <value>file:/data/hdfs/data</value>  </property>
  <!-- SecondaryNameNode的端口号，默认端口号是50090 -->  <property>    <name>dfs.namenode.secondary.http-address</name>    <value>hadoop1:50090</value>  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/mapred-site.xml,默认不存在，需要自建
```
<configuration>
  <!-- 第三方MapReduce框架，我们这里使用的yarn -->
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <!-- MapReduce JobHistory Server的IPC通信地址，默认端口号是10020 -->
  <property>     <name>mapreduce.jobhistory.address</name>     <value>hadoop1:10020</value>  </property>
  <!-- MapReduce JobHistory Server的Web服务器访问地址，默认端口号是19888 -->  <property>     <name>mapreduce.jobhistory.webapp.address</name>     <value>hadoop1:19888</value>  </property>
  <!-- MapReduce已完成作业信息 -->
  <property>
    <name>mapreduce.jobhistory.done-dir</name>
    <value>/data/history/done</value>
  </property>
  <!-- MapReduce正在运行作业信息 -->
  <property>
    <name>mapreduce.jobhistory.intermediate-done-dir</name>
    <value>/data/history/done_intermediate</value>
  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/yarn-site.xml
```
<configuration>
  <!-- 为MapReduce设置洗牌服务 -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>    <value>org.apache.hadoop.mapred.ShuffleHandler</value>  </property>
  <!-- NodeManager与ResourceManager通信的接口地址，默认端口是8032 -->
  <property>    <name>yarn.resourcemanager.address</name>    <value>hadoop1:8032</value>  </property>
  <!-- NodeManger需要知道ResourceManager主机的scheduler调度服务接口地址，默认端口是8030 -->  <property>    <name>yarn.resourcemanager.scheduler.address</name>    <value>hadoop1:8030</value>  </property>
  <!-- NodeManager需要向ResourceManager报告任务运行状态供Resouce跟踪，因此NodeManager节点主机需要知道ResourceManager主机的tracker接口地址，默认端口是8031 -->  <property>    <name>yarn.resourcemanager.resource-tracker.address</name>    <value>hadoop1:8031</value>  </property>
  <!-- resourcemanager.admin，默认端口是8033 -->  <property>    <name>yarn.resourcemanager.admin.address</name>    <value>hadoop1:8033</value>  </property>
  <!-- 各个task的资源调度及运行状况通过通过该web界面访问，默认端口是8088 -->  <property>    <name>yarn.resourcemanager.webapp.address</name>    <value>hadoop1:8088</value>  </property>
</configuration>
```

##### 配置slaves节点，修改HADOOP_HOME/etc/hadoop/slaves

如果slaves配置中也添加Hadoop1节点，那么Hadoop1节点就既是namenode，又是datanode，这里没有这么配置，所以Hadoop1节点只是namenode，所以下面启动Hadoop1的服务之后，jps查看只有namenode服务器而没有datanode服务

```
Hadoop2
Hadoop3
```

##### 配置主机名/etc/hosts

这里Hadoop1是namenode，Hadoop2和Hadoop3是datanode

```
192.168.1.119   hadoop1192.168.1.150   hadoop2192.168.1.149   hadoop3
```

##### 配置SSH免密码登录

在Hadoop1节点中生成新的SSH Key，并且将新生成的SSH Key添加到Hadoop1，2，3的authorized_keys免密码访问的配置中

```
# Hadoop1中执行
$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys

# Hadoop2，3中执行
$ scp root@Hadoop1:~/.ssh/id_dsa.pub  ~/.ssh/master_dsa.pub
$ cat ~/.ssh/master_dsa.pub >> ~/.ssh/authorized_keys

# 在Hadoop1中测试是否可以免密码登录Hadoop1，2，3（第一次应该只需要输入yes）
$ ssh root@Hadoop1
$ ssh root@Hadoop2
$ ssh root@Hadoop3
```

##### 配置好Hadoop1之后，将Hadoop1的配置copy到Hadoop2和Hadoop3
```
# 在Hadoop1中执行
$ scp -r /data/hadoop-2.7.1 root@Hadoop2:/data/
$ scp -r /data/hadoop-2.7.1 root@Hadoop3:/data/
```

##### 启动服务

```
# 初始化namenode
$ ./bin/hdfs namenode -format

# 初始化好namenode后，hadoop会自动建好对应hdfs-site.xml的namenode配置的文件路径
$ ll /data/hdfs/name/current/total 24drwxrwxr-x 2 yunyu yunyu 4096 Sep 10 18:07 ./drwxrwxr-x 3 yunyu yunyu 4096 Sep 10 18:07 ../-rw-rw-r-- 1 yunyu yunyu  352 Sep 10 18:07 fsimage_0000000000000000000-rw-rw-r-- 1 yunyu yunyu   62 Sep 10 18:07 fsimage_0000000000000000000.md5-rw-rw-r-- 1 yunyu yunyu    2 Sep 10 18:07 seen_txid-rw-rw-r-- 1 yunyu yunyu  202 Sep 10 18:07 VERSION

# 启动hdfs服务
$ ./sbin/start-dfs.sh
Starting namenodes on [hadoop1]hadoop1: starting namenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-namenode-ubuntu.outhadoop2: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.outhadoop3: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.outStarting secondary namenodes [hadoop1]hadoop1: starting secondarynamenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-secondarynamenode-ubuntu.out

# 使用jps检查启动的服务，可以看到NameNode和SecondaryNameNode已经启动
$ jps20379 SecondaryNameNode20570 Jps20106 NameNode

# 这时候在Hadoop2和Hadoop3节点上使用jps查看，DataNode已经启动
$ jps16392 Jps16024 DataNode

# 在Hadoop2和Hadoop3节点上，也会自动建好对应hdfs-site.xml的datanode配置的文件路径
$ ll /data/hdfs/data/current/total 16drwxrwxr-x 3 yunyu yunyu 4096 Sep 10 18:10 ./drwx------ 3 yunyu yunyu 4096 Sep 10 18:10 ../drwx------ 4 yunyu yunyu 4096 Sep 10 18:10 BP-1965589257-127.0.1.1-1473502067891/-rw-rw-r-- 1 yunyu yunyu  229 Sep 10 18:10 VERSION

# 启动yarn服务
$ ./sbin/start-yarn.sh
starting yarn daemonsstarting resourcemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-resourcemanager-ubuntu.outhadoop3: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.outhadoop2: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.out

# 使用jps检查启动的服务，可以看到ResourceManager已经启动
$ jps21653 Jps20379 SecondaryNameNode20106 NameNode21310 ResourceManager

# 这时候在Hadoop2和Hadoop3节点上使用jps查看，NodeManager已经启动
$ jps16946 NodeManager17235 Jps16024 DataNode

# 启动jobhistory服务，默认jobhistory在使用start-all.sh是不启动的，所以即使使用start-all.sh也要手动启动jobhistory服务
$ ./sbin/mr-jobhistory-daemon.sh start historyserver
starting historyserver, logging to /data/hadoop-2.7.1/logs/mapred-yunyu-historyserver-ubuntu.out

# 使用jps检查启动的服务，可以看到JobHistoryServer已经启动
$ jps21937 Jps20379 SecondaryNameNode20106 NameNode21863 JobHistoryServer21310 ResourceManager
```

注意：使用start-all.sh启动已经不再被推荐使用，所以这里使用的是Hadoop推荐的分开启动，分别启动start-dfs.sh和start-yarn.sh，所以看一些比较就的Hadoop版本安装的文章可能会用start-all.sh启动

##### 停止服务

```
# 停止hdfs服务
$ ./sbin/stop-dfs.sh

# 停止yarn服务
$ ./sbin/stop-yarn.sh

# 停止jobhistory服务
$ ./sbin/mr-jobhistory-daemon.sh stop historyserver
```

##### 验证Hadoop集群的Web服务

- 验证NameNode的Web服务能访问，浏览器访问http://192.168.1.119:50070
![](http://img.blog.csdn.net/20160910184234348?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 验证ResourceManager的Web服务能访问，浏览器访问http://192.168.1.119:8088
![](http://img.blog.csdn.net/20160910184156566?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 验证MapReduce JobHistory Server的Web服务能访问，浏览器访问http://192.168.1.119:19888
![](http://img.blog.csdn.net/20160910184251708?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 验证HDFS文件系统
```
# 查看根目录下的文件
$ hdfs dfs -ls /Found 1 itemsdrwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /data

# 创建temp目录
$ hdfs dfs -mkdir /temp

# 再次查看根目录下的文件，可以看到temp目录
$ hdfs dfs -ls /Found 2 itemsdrwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /datadrwxr-xr-x   - yunyu supergroup          0 2016-09-10 03:45 /temp

# 可以查看之前mapred-site.xml中配置的mapreduce作业执行中的目录和作业已完成的目录
$ hdfs dfs -ls /data/history/Found 2 itemsdrwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /data/history/donedrwxrwxrwt   - yunyu supergroup          0 2016-09-10 03:15 /data/history/done_intermediate
```

### 需要注意的地方

网上一些Hadoop集群安装相关文章中，有一部分还是Hadoop老版本的配置，所以有些迷惑，像JobTracker，TaskTracker这些概念是Hadoop老版本才有的，新版本中使用ResourceManager和NodeManager替代了他们。后续的章节会详细的介绍Hadoop的相关原理以及新老版本的区别。


参考文章：

- http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-common/ClusterSetup.html
- http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-common/core-default.xml
- http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml
- http://hadoop.apache.org/docs/r2.7.1/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml
- http://hadoop.apache.org/docs/r2.7.1/hadoop-yarn/hadoop-yarn-common/yarn-default.xml
- http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#secondarynamenode
- http://www.cnblogs.com/luogankun/p/4019303.html
- http://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-yarn/#_3.1_hadoop_0.23.0
- http://blog.chinaunix.net/uid-25266990-id-3900239.html
- http://blog.csdn.net/jxnu_xiaobing/article/details/46931693
- http://www.cnblogs.com/liuling/archive/2013/06/16/2013-6-16-01.html