---
title: "Docker实战（十四）Docker安装Hadoop环境"
date: 2016-06-20 23:42:27
tags: [Docker命令, Dockerfile, Hadoop]
categories: [Docker]
---

##### Hadoop安装

```
# 需要提前安装ssh
$ apt-get install ssh
$ apt-get install openssh-server

# 设置免密码登录，生成私钥和公钥
$ ssh-keygen -t rsa -P ""
$ cat ~/.ssh/id_rsa.put >> ~/.ssh/authorized_keys

# 测试是否可以免密码登录
$ ssh localhost

# 下载Hadoop
$ curl -O http://mirrors.cnnic.cn/apache/hadoop/common/hadoop-2.7.1/hadoop-2.7.1.tar.gz

# 解压Hadoop压缩包
$ tar -xf hadoop-2.7.1.tar.gz
```

##### Dockerfile文件
```
############################################
# version : birdben/hadoop:v1
# desc : 当前版本安装的hadoop
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 hadoop 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV HADOOP_HOME /software/hadoop-2.7.1
ENV PATH ${HADOOP_HOME}/bin:$PATH

# 复制 hadoop-2.7.1 文件到镜像中（hadoop-2.7.1 文件夹要和Dockerfile文件在同一路径）
ADD hadoop-2.7.1 /software/hadoop-2.7.1

# 创建HDFS存储路径
RUN mkdir -p $HADOOP_HOME/hdfs/name
RUN mkdir -p $HADOOP_HOME/hdfs/data

# 授权HADOOP_HOME路径给admin用户
RUN sudo chown -R admin /software/hadoop-2.7.1

# 容器需要开放Hadoop 9000端口
EXPOSE 9000

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/hadoop/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

# 暂时没用使用supervsord启动Hadoop，因为没研究明白如何启动的时候不需要ssh输入密码，即使设置了ENV DEBIAN_FRONTEND noninteractive所有的操作的是非交互的也不好用，所以只能启动Docker容器之后手动再启动Hadoop
# [program:hadoop]
# command=$HADOOP_HOME/bin/hdfs namenode -format
# command=$HADOOP_HOME/sbin/start_all.sh
```

##### 配置HADOOP_HOME/etc/hadoop/hadoop-env.sh，添加以下内容

```
export JAVA_HOME=/software/jdk7
export PATH=$PATH:/software/hadoop-2.7.1/bin
```

##### 配置HADOOP_HOME/etc/hadoop/yarn-env.sh，添加以下内容

```
export JAVA_HOME=/software/jdk7
```

##### 配置HADOOP_HOME/etc/hadoop/core-site.xml，这里的Ben是我Docker容器的hostname，读者自行把Ben替换成自己的hostname（下同）

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://Ben:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>file:/software/hadoop-2.7.1/tmp</value>
  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/hdfs-site.xml

```
<configuration>
  <property>
    <name>dfs.datanode.ipc.address</name>
    <value>Ben:50020</value>
  </property>
  <property>
    <name>dfs.datanode.http.address</name>
    <value>Ben:50075</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/software/hadoop-2.7.1/hdfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/software/hadoop-2.7.1/hdfs/data</value>
  </property>
  <property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>Ben:50090</value>
  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/mapred-site.xml,默认不存在，需要自建

```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

##### 配置HADOOP_HOME/etc/hadoop/yarn-site.xml

```
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```


##### 控制台终端

```
# 构建镜像
$ docker build -t="birdben/hadoop:v1" .
# 执行已经构件好的镜像，挂载在宿主机器的存储路径也不同，-h设置hostname（即上面所说的Docker容器的hostname【Ben】），Hadoop配置文件需要使用
$ docker run -h Ben -p 9999:22 -p 9000:9000 -p 8088:8088 -p 50020:50020 -p 50070:50070 -p 50075:50075 -p 50090:50090 -t -i 'birdben/hadoop:v1'
```

##### 用SSH登录admin账号启动Hadoop并且查看JPS

```
# 此时可以通过ssh远程连接Docker容器了
$ ssh root@10.211.55.4 -p 9999

# 先初始化namenode
admin@Ben:/$ cd /software/hadoop-2.7.1/bin
admin@Ben:/software/hadoop-2.7.1/sbin$ ./hdfs namenode -format

# 看到类似下面的信息就说明初始化namenode成功了
16/06/06 03:27:57 INFO namenode.FSNamesystem: Retry cache will use 0.03 of total heap and retry cache entry expiry time is 600000 millis
16/06/06 03:27:57 INFO util.GSet: Computing capacity for map NameNodeRetryCache
16/06/06 03:27:57 INFO util.GSet: VM type       = 64-bit
16/06/06 03:27:57 INFO util.GSet: 0.029999999329447746% max memory 889 MB = 273.1 KB
16/06/06 03:27:57 INFO util.GSet: capacity      = 2^15 = 32768 entries
16/06/06 03:27:57 INFO namenode.FSImage: Allocated new BlockPoolId: BP-581626940-172.17.0.2-1465183677567
16/06/06 03:27:57 INFO common.Storage: Storage directory /software/hadoop-2.7.1/hdfs/name has been successfully formatted.
16/06/06 03:27:57 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
16/06/06 03:27:57 INFO util.ExitUtil: Exiting with status 0
16/06/06 03:27:57 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at Ben/172.17.0.2
************************************************************/

# 启动所有服务
admin@Ben:/$ cd /software/hadoop-2.7.1/sbin
admin@Ben:/software/hadoop-2.7.1/sbin$ ./start-all.sh

# 下面的过程就是启动Hadoop的过程，但是始终需要输入ssh的密码，不知道为啥
This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
Starting namenodes on [Ben]
Ben: Could not create directory '/home/admin/.ssh'.
The authenticity of host 'ben (172.17.0.2)' can't be established.
ECDSA key fingerprint is ae:ec:44:6d:64:48:e8:34:17:33:cf:86:07:20:e3:4d.
Are you sure you want to continue connecting (yes/no)? yes
Ben: Failed to add the host to the list of known hosts (/home/admin/.ssh/known_hosts).
admin@ben's password:
Ben: Could not chdir to home directory /home/admin: No such file or directory
Ben: starting namenode, logging to /software/hadoop-2.7.1/logs/hadoop-admin-namenode-Ben.out
localhost: Could not create directory '/home/admin/.ssh'.
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is ae:ec:44:6d:64:48:e8:34:17:33:cf:86:07:20:e3:4d.
Are you sure you want to continue connecting (yes/no)? yes
localhost: Failed to add the host to the list of known hosts (/home/admin/.ssh/known_hosts).
admin@localhost's password:
localhost: Could not chdir to home directory /home/admin: No such file or directory
localhost: starting datanode, logging to /software/hadoop-2.7.1/logs/hadoop-admin-datanode-Ben.out
Starting secondary namenodes [Ben]
Ben: Could not create directory '/home/admin/.ssh'.
The authenticity of host 'ben (172.17.0.2)' can't be established.
ECDSA key fingerprint is ae:ec:44:6d:64:48:e8:34:17:33:cf:86:07:20:e3:4d.
Are you sure you want to continue connecting (yes/no)? yes
Ben: Failed to add the host to the list of known hosts (/home/admin/.ssh/known_hosts).
admin@ben's password:
Ben: Could not chdir to home directory /home/admin: No such file or directory
Ben: starting secondarynamenode, logging to /software/hadoop-2.7.1/logs/hadoop-admin-secondarynamenode-Ben.out
starting yarn daemons
starting resourcemanager, logging to /software/hadoop-2.7.1/logs/yarn-admin-resourcemanager-Ben.out
localhost: Could not create directory '/home/admin/.ssh'.
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is ae:ec:44:6d:64:48:e8:34:17:33:cf:86:07:20:e3:4d.
Are you sure you want to continue connecting (yes/no)? yes
localhost: Failed to add the host to the list of known hosts (/home/admin/.ssh/known_hosts).
admin@localhost's password:
localhost: Could not chdir to home directory /home/admin: No such file or directory
localhost: starting nodemanager, logging to /software/hadoop-2.7.1/logs/yarn-admin-nodemanager-Ben.out

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
```

##### 在Ubuntu宿主机浏览器查看

```
# 查看namenode状态：
http://127.0.0.1:50070/

# 查看yarn状态：
http://127.0.0.1:8088/
```


参考文章：

- http://my.oschina.net/adrianlynn/blog/505475?fromerr=n1qLgsaT
- http://my.oschina.net/u/204498/blog/519789?fromerr=9ZYyEtZm
- http://blog.csdn.net/zhaogezhuoyuezhao/article/details/8426910
- http://tashan10.com/yong-dockerda-jian-hadoopwei-fen-bu-shi-ji-qun/
- 集群
- http://blog.csdn.net/peace1213/article/details/51334508