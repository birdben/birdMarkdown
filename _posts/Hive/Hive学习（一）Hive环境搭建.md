---
title: "Hive学习（一）Hive环境搭建"
date: 2016-09-18 16:48:04
tags: [Hive, HDFS]
categories: [Hadoop]
---

Hive必须运行在Hadoop之上，则需要先安装Hadoop环境，而且还需要MySQL数据库，具体Hadoop安装请参考Hadoop系列文章

### Hive环境安装

```
# 下载Hive
$ wget http://apache.mirrors.ionfish.org/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz

# 解压Hive压缩包
$ tar -zxvf apache-hive-1.2.1-bin.tar.gz

# 下载MySQL驱动包
$ wget http://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.38.tar.gz

# 解压MySQL驱动压缩包
$ tar -zxvf mysql-connector-java-5.1.38.tar.gz
```

### Hive相关的配置文件

注意：以下配置请根据自己的实际环境修改

##### 配置环境变量/etc/profile
```HIVE_HOME=/usr/local/hiveexport HIVE_HOME
HIVE_JARS=$HIVE_HOME/libexport HIVE_JARS
PATH=$JAVA_HOME/bin:$HIVE_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$MAVEN_HOME/bin:$PATHexport PATH
```

##### 配置HIVE_HOME/conf/hive-env.sh（默认不存在，将hive-env.sh.template复制并改名为hive-env.sh）

```
# 这里使用此路径是因为安装Hadoop环境的时候，设置了环境变量PATH
HADOOP_HOME=/usr/local/hadoop
```

##### 配置HIVE_HOME/conf/hive-log4j.properties（默认不存在，将hive-log4j.properties.template复制并改名为hive-log4j.properties）

这里使用默认配置即可，不需要修改

##### 配置HIVE_HOME/conf/hdfs-site.xml（默认不存在，将hive-default.xml.template复制并改名为hive-site.xml）

这里的Hadoop1是我们Hadoop集群的namenode主机的hostname，mysql安装在另外一台机器10.10.1.46上

```
<?xml version="1.0" encoding="UTF-8" standalone="no"?><?xml-stylesheet type="text/xsl" href="configuration.xsl"?><configuration>   <property>       <!-- metastore我的mysql不是在该server上，是在另一台Docker镜像中 -->		<name>hive.metastore.local</name>		<value>false</value>   </property>   <property>		<!-- mysql服务的ip和端口号 -->		<name>javax.jdo.option.ConnectionURL</name>		<value>jdbc:mysql://10.10.1.46:3306/hive</value>   </property>   <property>		<name>javax.jdo.option.ConnectionDriveName</name>		<value>com.mysql.jdbc.Driver</value>   </property>   <property>		<name>javax.jdo.option.ConnectionUserName</name>		<value>root</value>   </property>   <property>		<name>javax.jdo.option.ConnectionPassword</name>		<value>root</value>   </property>   <property>		<!-- hive的仓库目录，需要在HDFS上创建，并修改权限 -->		<name>hive.metastore.warehouse.dir</name>		<value>/hive/warehouse</value>   </property>   <property>		<!-- 运行hive得主机地址及端口，即本机ip和端口号，启动metastore服务 -->		<name>hive.metastore.uris</name>		<value>thrift://Hadoop1:9083</value>   </property></configuration>
```

##### 控制台终端

```
# 初始化namenode
#（这一步根据自己的实际情况选择是否初始化，如果初始化过了就不需要再初始化了）
$ ./bin/hdfs namenode -format

# 启动hdfs服务
$ ./sbin/start-dfs.sh
Starting namenodes on [hadoop1]hadoop1: starting namenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-namenode-ubuntu.outhadoop2: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.outhadoop3: starting datanode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-datanode-ubuntu.outStarting secondary namenodes [hadoop1]hadoop1: starting secondarynamenode, logging to /data/hadoop-2.7.1/logs/hadoop-yunyu-secondarynamenode-ubuntu.out

# 启动yarn服务
$ ./sbin/start-yarn.sh
starting yarn daemonsstarting resourcemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-resourcemanager-ubuntu.outhadoop3: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.outhadoop2: starting nodemanager, logging to /data/hadoop-2.7.1/logs/yarn-yunyu-nodemanager-ubuntu.out

# 进入hive安装目录
$ cd /data/hive-1.2.1

# 启动metastore
# 注意：启动metastore之前一定要检查hive-site.xml配置文件中配置的mysql数据库地址10.10.1.46中是否有配置的hive数据库，如果没有启动会报错，需要事先创建好空的数据库，启动metastore后会自动初始化hive的元数据表
$ ./bin/hive --service metastore &

# 启动的时候可能会遇到下面的错误，是因为没有找到mysql驱动包
Caused by: java.sql.SQLException: No suitable driver found for jdbc:mysql://10.10.1.46:3306/hive	at java.sql.DriverManager.getConnection(DriverManager.java:596)	at java.sql.DriverManager.getConnection(DriverManager.java:187)	at com.jolbox.bonecp.BoneCP.obtainRawInternalConnection(BoneCP.java:361)	at com.jolbox.bonecp.BoneCP.<init>(BoneCP.java:416)	... 48 more

# 把下载的mysql驱动包copy到hive/lib目录下重启即可
$ cp mysql-connector-java-5.1.38-bin.jar /data/hive-1.2.1/lib/

# 启动hiveserver2
$ ./bin/hive --service hiveserver2 &

# 此时重新启动hive shell，就可以成功登录hive了
$ ./bin/hive shell
hive>
hive> show databases;
OK
default
Time taken: 1.323 seconds, Fetched: 1 row(s)

# 注意：这里使用的MySQL的root账号需要处理更改密码和远程登录授权问题，所以这里没有涉及这些问题，具体设置可以参考之前的Docker安装MySQL镜像的文章

# 我们需要预先在mysql中创建一个hive的数据库，因为hive-site.xml是连接到这个hive数据库的，所有的hive元数据都是存在这个hive数据库中的
# 我们在hive中创建新的数据库和表来验证hive的元数据都存储在mysql了

# 在hive中创建一个新的数据库test_hive，test_hive这个数据库会对应mysql中的hive数据库中的DBS表中的一条记录
hive> CREATE DATABASE test_hive;

# 在hive中创建一个新的表test_person，test_person这个表会对应mysql中的hive数据库中的TBLS表中的一条记录
hive> USE test_hive;
hive> CREATE TABLE test_person (id INT,name string) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';


# 在hive创建表的时候可能会遇到如下问题，是因为MySQL数据库字符集设置的utf-8导致的
# Specified key was too long; max key length is 767 bytes
# 修改MySQL的hive数据库的字符集为latin1就好用了
$ alter database hive character set latin1;
# 参考：http://blog.163.com/zhangjie_0303/blog/static/990827062013112623615941/

# test_person.txt
1	John
2	Ben
3	Allen
4	Jimmy
5	Will
6	Jackson

# 导入数据到test_person.txt到test_person表
hive> LOAD DATA LOCAL INPATH '/data/test_person.txt' OVERWRITE INTO TABLE test_person;
Loading data to table test_hive.test_person
Table test_hive.test_person stats: [numFiles=1, numRows=0, totalSize=45, rawDataSize=0]
OK
Time taken: 2.885 seconds

# 查看test_person表数据
hive> select * from test_person;
OK
1	John
2	Ben
3	Allen
4	Jimmy
5	Will
6	Jackson
Time taken: 0.7 seconds, Fetched: 6 row(s)

# 查看test_hive数据库在HDFS中存储的目录
$ cd /data/hadoop-2.7.1/bin

# 查看HDFS中/hive/warehouse目录下的所有文件，此目录是在hive-site.xml中hive.metastore.warehouse.dir参数配置的路径/hive/warehouse
$ ./bin/hdfs dfs -ls /hive/warehouse/
Found 1 items
drwxr-xr-x   - admin supergroup          0 2016-06-25 11:39 /hive/warehouse/test_hive.db

# 查看test_person表在HDFS中存储的目录
$ ./bin/hdfs dfs -ls /hive/warehouse/test_hive.db/
Found 1 items
drwxr-xr-x   - admin supergroup          0 2016-06-25 11:52 /hive/warehouse/test_hive.db/test_person

# 在深入一层就能看到我们导入的文件test_person.txt了
$ ./bin/hdfs dfs -ls /hive/warehouse/test_hive.db/test_person/
Found 1 items
-rwxr-xr-x   3 admin supergroup         45 2016-06-25 11:52 /hive/warehouse/test_hive.db/test_person/test_person.txt

# 查看test_person.txt文件里的内容，就是我们导入的内容
$ ./bin/hdfs dfs -cat /hive/warehouse/test_hive.db/test_person/test_person.txt
1	John
2	Ben
3	Allen
4	Jimmy
5	Will
6	Jackson
```


参考文章：

- http://my.oschina.net/u/204498/blog/522772
- http://blog.fens.me/hadoop-hive-intro/
- http://www.mincoder.com/article/5809.shtml
- http://blog.163.com/zhangjie_0303/blog/static/990827062013112623615941/