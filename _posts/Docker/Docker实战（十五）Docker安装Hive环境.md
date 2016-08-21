---
title: "Docker实战（十五）Docker安装Hive环境"
date: 2016-06-25 20:30:28
tags: [Docker命令, Dockerfile, Hive]
categories: [Docker]
---

##### Hive安装

```
# Hive必须运行在Hadoop之上，则需要先安装Hadoop环境，而且还需要MySQL数据库，具体Hadoop安装请参考上一篇文章，我们这里继承上一篇已经安装好的Hadoop镜像

# 下载Hive
$ wget http://apache.mirrors.ionfish.org/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz

# 解压Hive压缩包
$ tar -zxvf apache-hive-1.2.1-bin.tar.gz

# 下载MySQL驱动包
$ wget http://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.38.tar.gz

# 解压MySQL驱动压缩包
$ tar -zxvf mysql-connector-java-5.1.38.tar.gz

# 需要修改下面Hive相关的配置文件
```

##### Dockerfile文件
```
############################################
# version : birdben/hive:v1
# desc : 当前版本安装的hive
############################################
# 设置继承自我们创建的 hadoop 镜像
FROM birdben/hadoop:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 复制 hive-1.2.1 文件到镜像中（hive-1.2.1 文件夹要和Dockerfile文件在同一路径），这里直接把上一篇Hadoop环境直接用上了
ADD apache-hive-1.2.1-bin /software/hive-1.2.1

# 复制MySQL的驱动包到镜像中
COPY mysql-connector-java-5.1.38/mysql-connector-java-5.1.38-bin.jar /software/hive-1.2.1/lib/mysql-connector-java-5.1.38-bin.jar

# 授权HIVE_HOME路径给admin用户
RUN sudo chown -R admin /software/hive-1.2.1

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/hive/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:metastore]
command=/software/hive-1.2.1/bin/hive --service metastore

[program:hiveserver2]
command=/software/hive-1.2.1/bin/hive --service hiveserver2

```

##### 配置HIVE_HOME/conf/hive-env.sh（默认不存在，将hive-env.sh.template复制并改名为hive-env.sh）

```
HADOOP_HOME=/software/hadoop-2.7.1
```

##### 配置HIVE_HOME/conf/hive-log4j.properties（默认不存在，将hive-log4j.properties.template复制并改名为hive-log4j.properties）

##### 配置HIVE_HOME/conf/hdfs-site.xml（默认不存在，将hive-default.xml.template复制并改名为hive-site.xml）

```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
	<property>
    <!-- metastore我的mysql不是在该server上，是在另一台Docker镜像中 -->
    <name>hive.metastore.local</name>
    <value>false</value>
  </property>
	<property>
    <!-- mysql服务的ip和端口号 -->
		<name>javax.jdo.option.ConnectionURL</name>
		<value>jdbc:mysql://172.17.0.2:3306/hive</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionDriveName</name>
		<value>com.mysql.jdbc.Driver</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionUserName</name>
		<value>root</value>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionPassword</name>
		<value>root</value>
	</property>
	<property>
    <!-- hive的仓库目录，需要在HDFS上创建，并修改权限 -->
		<name>hive.metastore.warehouse.dir</name>
		<value>/hive/warehouse</value>
	</property>
	<property>
    <!-- 运行hive得主机地址及端口，即本机ip和端口号，启动metastore服务 -->
	  <name>hive.metastore.uris</name>
    <value>thrift://Ben:9083</value>
	</property>
</configuration>
```

##### 控制台终端

```
# 先启动MySQL的Docker容器
$ sudo docker run -p 9999:22 -p 3306:3306 -t -i -v /docker/mysql/data:/software/mysql-5.6.22/data birdben/mysql:v1

# 然后查看MySQL的Docker容器的IP地址，Hive连接MySQL的IP地址需要配置这个IP
$ sudo docker inspect --format '{{.NetworkSettings.IPAddress}}' 34b5ac61b8bd
$ 172.17.0.2

# 构建镜像
$ docker build -t="birdben/hive:v1" .
# 执行已经构件好的镜像，挂载在宿主机器的存储路径也不同，-h设置hostname，Hadoop配置文件需要使用
$ docker run -h Ben -p 9998:22 -p 9083:9083 -p 9000:9000 -p 8088:8088 -p 50020:50020 -p 50070:50070 -p 50075:50075 -p 50090:50090 -t -i 'birdben/hive:v1'


# 然后直接通过ssh远程连接使用admin账号远程登录Hive的Docker容器
$ ssh root@10.211.55.4 -p 9998

# 启动hive shell，发现如下的错误，原因是还没有启动Hadoop，所以hive连接Hadoop时，报错java.net.ConnectException: Call From Ben/172.17.0.3 to Ben:9000 failed on connection exception: java.net.ConnectException: Connection refused;

$ admin@Ben:/$ cd /software/hive-1.2.1/bin
$ admin@Ben:/$ ./hive shell
16/06/25 10:48:40 WARN conf.HiveConf: HiveConf of name hive.metastore.local does not exist

Logging initialized using configuration in file:/software/hive-1.2.1/conf/hive-log4j.properties
Exception in thread "main" java.lang.RuntimeException: java.net.ConnectException: Call From Ben/172.17.0.3 to Ben:9000 failed on connection exception: java.net.ConnectException: Connection refused; For more details see:  http://wiki.apache.org/hadoop/ConnectionRefused

# 所以我们先启动Hadoop
# 先初始化namenode
$ admin@Ben:/$ cd /software/hadoop-2.7.1/bin
$ admin@Ben:/software/hadoop-2.7.1/sbin$ ./hdfs namenode -format

# 启动所有服务
admin@Ben:/$ cd /software/hadoop-2.7.1/sbin
admin@Ben:/software/hadoop-2.7.1/sbin$ ./start-all.sh

# 执行jps命令之前需要先把$JAVA_HOME/bin添加到PATH环境变量
admin@Ben:/$ export PATH=/software/jdk7/bin/:$PATH

# 执行jps命令如果能看到下面的6个进程就说明Hadoop启动的没有问题
admin@Ben:/software/hadoop-2.7.1/sbin$ jps
985 Jps
282 DataNode
587 ResourceManager
436 SecondaryNameNode
166 NameNode
691 NodeManager

# 此时重新启动hive shell，就可以成功登录hive了
$ admin@Ben:/$ cd /software/hive-1.2.1/bin
$ admin@Ben:/$ ./hive shell
hive>
hive> show databases;
OK
default
Time taken: 1.323 seconds, Fetched: 1 row(s)

# 注意：这里我是使用了之前的MySQL的Docker镜像，因为之前的MySQL的Docker镜像已经处理了root账号的更改密码和远程登录授权问题，所以这里没有涉及这些问题，具体设置可以参考之前的Docker安装MySQL镜像的文章

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

# 用scp命令将本地test_person.txt文件传到hive的Docker容器中
$ scp -P 9998 /Users/ben/workspace_git/birdDocker/hive/test_person.txt admin@10.211.55.4:/software/hive-1.2.1/

# 导入数据到test_person.txt到test_person表
hive> LOAD DATA LOCAL INPATH '/software/hive-1.2.1/test_person.txt' OVERWRITE INTO TABLE test_person;
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
$ admin@Ben:$ cd /software/hadoop-2.7.1/bin

# 查看HDFS中/hive/warehouse目录下的所有文件，此目录是在hive-site.xml中hive.metastore.warehouse.dir参数配置的路径/hive/warehouse
$ admin@Ben:/software/hadoop-2.7.1/bin$ ./hdfs dfs -ls /hive/warehouse/
Found 1 items
drwxr-xr-x   - admin supergroup          0 2016-06-25 11:39 /hive/warehouse/test_hive.db

# 查看test_person表在HDFS中存储的目录
$ admin@Ben:/software/hadoop-2.7.1/bin$ ./hdfs dfs -ls /hive/warehouse/test_hive.db/
Found 1 items
drwxr-xr-x   - admin supergroup          0 2016-06-25 11:52 /hive/warehouse/test_hive.db/test_person

# 在深入一层就能看到我们导入的文件test_person.txt了
$ admin@Ben:/software/hadoop-2.7.1/bin$ ./hdfs dfs -ls /hive/warehouse/test_hive.db/test_person/
Found 1 items
-rwxr-xr-x   3 admin supergroup         45 2016-06-25 11:52 /hive/warehouse/test_hive.db/test_person/test_person.txt

# 查看test_person.txt文件里的内容，就是我们导入的内容
$ admin@Ben:/software/hadoop-2.7.1/bin$ ./hdfs dfs -cat /hive/warehouse/test_hive.db/test_person/test_person.txt
1	John
2	Ben
3	Allen
4	Jimmy
5	Will
6	Jackson

# OK，大功告成了，可以回家了 ^_^
```
##### test_hive数据库在MySQL的hive数据库中的DBS表中的一条记录
![DBS表](http://img.blog.csdn.net/20160625194151411?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### test_person表在MySQL的hive数据库中的TBLS表中的一条记录
![TBLS表](http://img.blog.csdn.net/20160625194314403?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### 元数据解析
在MySQL可以查看Hive元数据表，除了DBS和TBLS存储数据库和表的基本信息，其他表的说明见下表:

|MySQL的表			|说明
|------			|------
|BUCKETING_COLS	|Hive表CLUSTERED BY字段信息(字段名，字段序号)
|CDS				|	
|COLUMNS_V2		|存放表格的字段信息
|DATABASE_PARAMS	
|DBS				|存放hive所有数据库信息
|FUNCS	
|FUNC_RU	
|GLOBAL_PRIVS	
|PARTITIONS		|Hive表分区信息(创建时间，具体的分区)
|PARTITION_KEYS	|Hive分区表分区键(名称，类型，comment，序号)
|PARTITION_KEY_VALS	|Hive表分区名(键值，序号)
|PARTITION_PARAMS	
|PART_COL_STATS	
|ROLES	
|SDS				|所有hive表、表分区所对应的hdfs数据目录和数据格式。
|SD_PARAMS	
|SEQUENCE_TABLE	|SEQUENCE_TABLE表保存了hive对象的下一个可用ID，如’org.apache.hadoop.hive.metastore.model.MTable’, 21，则下一个新创建的hive表其TBL_ID就是21，同时SEQUENCE_TABLE表中271786被更新为26(这里每次都是+5)。同样，COLUMN，PARTITION等都有相应的记录
|SERDES			|Hive表序列化反序列化使用的类库信息
|SERDE_PARAMS		|序列化反序列化信息，如行分隔符、列分隔符、NULL的表示字符等
|SKEWED_COL_NAMES	
|SKEWED_COL_VALUE_LOC_MAP	
|SKEWED_STRING_LIST	
|SKEWED_STRING_LIST_VALUES	
|SKEWED_VALUES	
|SORT_COLS		|Hive表SORTED BY字段信息(字段名，sort类型，字段序号)
|TABLE_PARAMS		|表级属性，如是否外部表，表注释等
|TAB_COL_STATS	
|TBLS				|所有hive表的基本信息
|VERSION	


参考文章：

- http://my.oschina.net/u/204498/blog/522772
- http://blog.fens.me/hadoop-hive-intro/
- http://www.mincoder.com/article/5809.shtml
- http://blog.163.com/zhangjie_0303/blog/static/990827062013112623615941/