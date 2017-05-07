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
192.168.1.119   hadoop1
192.168.1.150   hadoop2
192.168.1.149   hadoop3
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
JAVA_HOME=/usr/local/java
export JAVA_HOME
HADOOP_HOME=/usr/local/hadoop
export HADOOP_HOME
PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export PATH
```

##### 配置HADOOP_HOME/etc/hadoop/hadoop-env.sh，添加以下内容
```
export JAVA_HOME=/usr/local/java
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
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
  <!-- 分布式文件系统数据块复制数，我们这里是Hadoop2和Hadoop3两个节点 -->
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <!-- DFS namenode存放name table的目录 -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/data/hdfs/name</value>
  </property>
  <!-- DFS datanode存放数据block的目录 -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/data/hdfs/data</value>
  </property>
  <!-- SecondaryNameNode的端口号，默认端口号是50090 -->
  <property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>hadoop1:50090</value>
  </property>
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
  <property>
     <name>mapreduce.jobhistory.address</name>
     <value>hadoop1:10020</value>
  </property>
  <!-- MapReduce JobHistory Server的Web服务器访问地址，默认端口号是19888 -->
  <property>
     <name>mapreduce.jobhistory.webapp.address</name>
     <value>hadoop1:19888</value>
  </property>
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
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <!-- NodeManager与ResourceManager通信的接口地址，默认端口是8032 -->
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>hadoop1:8032</value>
  </property>
  <!-- NodeManger需要知道ResourceManager主机的scheduler调度服务接口地址，默认端口是8030 -->
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>hadoop1:8030</value>
  </property>
  <!-- NodeManager需要向ResourceManager报告任务运行状态供Resouce跟踪，因此NodeManager节点主机需要知道ResourceManager主机的tracker接口地址，默认端口是8031 -->
  <property>
    <name>yarn.resourcemanager.resource-tracker.address</name>
    <value>hadoop1:8031</value>
  </property>
  <!-- resourcemanager.admin，默认端口是8033 -->
  <property>
    <name>yarn.resourcemanager.admin.address</name>
    <value>hadoop1:8033</value>
  </property>
  <!-- 各个task的资源调度及运行状况通过通过该web界面访问，默认端口是8088 -->
  <property>
    <name>yarn.resourcemanager.webapp.address</name>
    <value>hadoop1:8088</value>
  </property>
</configuration>
```

##### 配置slaves节点，修改HADOOP_HOME/etc/hadoop/slaves

如果slaves配置中也添加Hadoop1节点，那么Hadoop1节点就既是namenode，又是datanode，这里没有这么配置，所以Hadoop1节点只是namenode，所以下面启动Hadoop1的服务之后，jps查看只有namenode服务器而没有datanode服务

```
Hadoop2
Hadoop3
```

##### 配置/etc/hostname

Hadoop1,2,3分别修改自己的/etc/hostname文件，如果这里不修改的话，后面使用Hive做离线查询会遇到问题，具体问题请参考《Hive学习（二）使用Hive进行离线分析日志》

```
hadoop1
```

##### 配置主机名/etc/hosts

这里Hadoop1是namenode，Hadoop2和Hadoop3是datanode

```
192.168.1.119   hadoop1
192.168.1.150   hadoop2
192.168.1.149   hadoop3
```

##### 配置SSH免密码登录

在Hadoop1节点中生成新的SSH Key，并且将新生成的SSH Key添加到Hadoop1，2，3的authorized_keys免密码访问的配置中

```
# 创建authorized_keys文件
$ vi ~/.ssh/authorized_keys

# 注意：这里authorized_keys文件的权限设置为600。（这点很重要，网没有设置600权限会导致登录失败）因为我这里用的root账户没有这个问题，但是如果用自己创建的其他hadoop账户，不设置600权限就会导致登录失败

# Hadoop1中执行
$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
# 将Hadoop1中的公钥复制进去
$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys

# Hadoop2，3中执行
$ scp root@Hadoop1:~/.ssh/id_dsa.pub  ~/.ssh/master_dsa.pub
# 将Hadoop2，3中的公钥复制进去
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
$ ll /data/hdfs/name/current/
total 24
drwxrwxr-x 2 yunyu yunyu 4096 Sep 10 18:07 ./
drwxrwxr-x 3 yunyu yunyu 4096 Sep 10 18:07 ../
-rw-rw-r-- 1 yunyu yunyu  352 Sep 10 18:07 fsimage_0000000000000000000
-rw-rw-r-- 1 yunyu yunyu   62 Sep 10 18:07 fsimage_0000000000000000000.md5
-rw-rw-r-- 1 yunyu yunyu    2 Sep 10 18:07 seen_txid
-rw-rw-r-- 1 yunyu yunyu  202 Sep 10 18:07 VERSION

# 启动hdfs服务
$ ./sbin/start-dfs.sh
Starting namenodes on [hadoop1]
hadoop1: starting namenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-namenode-ubuntu.out
hadoop2: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.out
hadoop3: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.out
Starting secondary namenodes [hadoop1]
hadoop1: starting secondarynamenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-secondarynamenode-ubuntu.out

# 使用jps检查启动的服务，可以看到NameNode和SecondaryNameNode已经启动
$ jps
20379 SecondaryNameNode
20570 Jps
20106 NameNode

# 这时候在Hadoop2和Hadoop3节点上使用jps查看，DataNode已经启动
$ jps
16392 Jps
16024 DataNode

# 在Hadoop2和Hadoop3节点上，也会自动建好对应hdfs-site.xml的datanode配置的文件路径
$ ll /data/hdfs/data/current/
total 16
drwxrwxr-x 3 yunyu yunyu 4096 Sep 10 18:10 ./
drwx------ 3 yunyu yunyu 4096 Sep 10 18:10 ../
drwx------ 4 yunyu yunyu 4096 Sep 10 18:10 BP-1965589257-127.0.1.1-1473502067891/
-rw-rw-r-- 1 yunyu yunyu  229 Sep 10 18:10 VERSION

# 启动yarn服务
$ ./sbin/start-yarn.sh
starting yarn daemons
starting resourcemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-resourcemanager-ubuntu.out
hadoop3: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.out
hadoop2: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.out

# 使用jps检查启动的服务，可以看到ResourceManager已经启动
$ jps
21653 Jps
20379 SecondaryNameNode
20106 NameNode
21310 ResourceManager

# 这时候在Hadoop2和Hadoop3节点上使用jps查看，NodeManager已经启动
$ jps
16946 NodeManager
17235 Jps
16024 DataNode

# 启动jobhistory服务，默认jobhistory在使用start-all.sh是不启动的，所以即使使用start-all.sh也要手动启动jobhistory服务
$ ./sbin/mr-jobhistory-daemon.sh start historyserver
starting historyserver, logging to /data/hadoop-2.7.1/logs/mapred-yunyu-historyserver-ubuntu.out

# 使用jps检查启动的服务，可以看到JobHistoryServer已经启动
$ jps
21937 Jps
20379 SecondaryNameNode
20106 NameNode
21863 JobHistoryServer
21310 ResourceManager
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
$ hdfs dfs -ls /
Found 1 items
drwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /data

# 创建temp目录
$ hdfs dfs -mkdir /temp

# 再次查看根目录下的文件，可以看到temp目录
$ hdfs dfs -ls /
Found 2 items
drwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /data
drwxr-xr-x   - yunyu supergroup          0 2016-09-10 03:45 /temp

# 可以查看之前mapred-site.xml中配置的mapreduce作业执行中的目录和作业已完成的目录
$ hdfs dfs -ls /data/history/
Found 2 items
drwxrwx---   - yunyu supergroup          0 2016-09-10 03:15 /data/history/done
drwxrwxrwt   - yunyu supergroup          0 2016-09-10 03:15 /data/history/done_intermediate
```

### 需要注意的地方

网上一些Hadoop集群安装相关文章中，有一部分还是Hadoop老版本的配置，所以有些迷惑，像JobTracker，TaskTracker这些概念是Hadoop老版本才有的，新版本中使用ResourceManager和NodeManager替代了他们。后续的章节会详细的介绍Hadoop的相关原理以及新老版本的区别。

最近好久没有用Hadoop了，突然要做日志持久化，居然本地的Hadoop集群环境起不来了，后来发现是自己启动方式错了，三个Hadoop节点只需要在NameNode执行start-dfs.sh和start-yarn.sh脚本，而我却分别在三个Hadoop节点都去做了启动操作，发现下面的提示信息才反应过来，真是太尴尬了。。

```
$ start-dfs.sh
Starting namenodes on [hadoop1]
yunyu@hadoop1's password: 
hadoop1: namenode running as process 9117. Stop it first.
```

### 使用HDFS默认端口号8020配置

修改core-site.xml配置文件如下（即把端口号去掉）

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop1</value>
  </property>
</configuration>
```

启动HDFS服务之后，分别在Hadoop1，2，3三台服务器上查看8020端口，发现HDFS默认使用的是8020端口

```
# 启动HDFS服务
$ ./sbin/start-dfs.sh

# Hadoop1中查看8020端口
$ lsof -i:8020
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    5112 yunyu  197u  IPv4  26041      0t0  TCP hadoop1:8020 (LISTEN)
java    5112 yunyu  207u  IPv4  27568      0t0  TCP hadoop1:8020->hadoop2:34867 (ESTABLISHED)
java    5112 yunyu  208u  IPv4  26096      0t0  TCP hadoop1:8020->hadoop3:59852 (ESTABLISHED)
java    5112 yunyu  209u  IPv4  29792      0t0  TCP hadoop1:8020->hadoop1:45542 (ESTABLISHED)
java    5383 yunyu  196u  IPv4  28826      0t0  TCP hadoop1:45542->hadoop1:8020 (ESTABLISHED)

# Hadoop2中查看8020端口
$ lsof -i:8020
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    4609 yunyu  234u  IPv4  24013      0t0  TCP hadoop2:34867->hadoop1:8020 (ESTABLISHED)

# Hadoop3中查看8020端口
$ lsof -i:8020
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    4452 yunyu  234u  IPv4  23413      0t0  TCP hadoop3:59852->hadoop1:8020 (ESTABLISHED)
```

访问HDFS集群的方式

```
# 访问本机的HDFS集群
hdfs dfs -ls /

# 可以指定host和port访问远程的HDFS集群（这里使用hostname和port访问本地集群）
hdfs dfs -ls hdfs://Hadoop1:8020/

# 如果使用的默认端口号8020，也可以不指定端口号访问
hdfs dfs -ls hdfs://Hadoop1/
```

### 解决Unable to load native-hadoop library for your platform

```
# 线上环境安装Hadoop的时候遇到下面的错误
$ start-dfs.sh
17/02/07 16:00:24 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Starting namenodes on [yy-logs-hdfs01]
yy-logs-hdfs01: starting namenode, logging to /usr/local/hadoop-2.7.1/logs/hadoop-hadoop-namenode-yy-logs-hdfs01.out
localhost: starting datanode, logging to /usr/local/hadoop-2.7.1/logs/hadoop-hadoop-datanode-yy-logs-hdfs01.out
Starting secondary namenodes [yy-logs-hdfs01]
yy-logs-hdfs01: starting secondarynamenode, logging to /usr/local/hadoop-2.7.1/logs/hadoop-hadoop-secondarynamenode-yy-logs-hdfs01.out
17/02/07 16:00:39 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable

# 于是百度Google查找原因，网上有人说是因为Apache提供的hadoop本地库是32位的，而在64位的服务器上就会有问题，因此需要自己编译64位的版本。
# 检查native库，发现果然是这个原因
$ hadoop checknative -a
17/02/07 16:06:34 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Native library checking:
hadoop:  false
zlib:    false
snappy:  false
lz4:     false
bzip2:   false
openssl: false
17/02/07 16:06:34 INFO util.ExitUtil: Exiting with status 1

# 从下面的地址下载Hadoop对应版本已经编译好的Native库，我这里下载的是hadoop-2.7.x版本的
# http://dl.bintray.com/sequenceiq/sequenceiq-bin/

# 将下载的Native库解压到$HADOOP_HOME下的lib和lib/native目录下
$ tar -xvf hadoop-native-64-2.7.0.tar -C /usr/local/hadoop/lib/
$ tar -xvf hadoop-native-64-2.7.0.tar -C /usr/local/hadoop/lib/native/

# 重新检查native库
$ hadoop checknative -a
17/02/07 16:26:56 WARN bzip2.Bzip2Factory: Failed to load/initialize native-bzip2 library system-native, will use pure-Java version
17/02/07 16:26:56 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library
Native library checking:
hadoop:  true /usr/local/hadoop-2.7.1/lib/libhadoop.so.1.0.0
zlib:    true /lib64/libz.so.1
snappy:  false
lz4:     true revision:99
bzip2:   false
openssl: true /usr/lib64/libcrypto.so
17/02/07 16:26:56 INFO util.ExitUtil: Exiting with status 1
```

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
- http://www.cnblogs.com/luogankun/p/4019303.html
- http://jacoxu.com/?p=961
- http://blog.csdn.net/jack85986370/article/details/51902871
- http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-common/NativeLibraries.html
- http://blog.csdn.net/liuxinghao/article/details/40121843