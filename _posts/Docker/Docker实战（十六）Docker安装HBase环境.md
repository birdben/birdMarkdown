---
title: "Docker实战（十六）Docker安装HBase环境"
date: 2016-07-01 00:05:19
tags: [Docker命令, Dockerfile, HBase]
categories: [Docker]
---

##### HBase安装

```
# 下载HBase
$ wget http://apache.fayea.com/hbase/0.98.19/hbase-0.98.19-hadoop2-bin.tar.gz

# 解压Hive压缩包
$ tar -zxvf hbase-0.98.19-hadoop2-bin.tar.gz

# 需要修改下面HBase相关的配置文件
```

##### Dockerfile文件
```
############################################
# version : birdben/hbase:v1
# desc : 当前版本安装的hbase
############################################
# 设置继承自我们创建的 jdk7 镜像
FROM birdben/jdk7:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 设置 hbase 的环境变量，若读者有其他的环境变量需要设置，也可以在这里添加。
ENV HBASE_HOME /software/hbase-0.98.19
ENV PATH ${HBASE_HOME}/bin:$PATH

# 复制 hbase-0.98.19 文件到镜像中（hbase-0.98.19 文件夹要和Dockerfile文件在同一路径），这里直接把上一篇Hadoop环境直接用上了
ADD hbase-0.98.19 /software/hbase-0.98.19

# 授权HBASE_HOME路径给admin用户
RUN sudo chown -R admin /software/hbase-0.98.19

# 容器需要开放HBase 60010端口
EXPOSE 60010

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]
```

##### Dockerfile源文件链接：

https://github.com/birdben/birdDocker/blob/master/hbase/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D

[program:hbase]
command=/software/hbase-0.98.19/bin/start-hbase.sh
```

##### 配置HBASE_HOME/conf/hbase-env.sh

```
export JAVA_HOME=/software/jdk7
export HBASE_CLASSPATH=/software/hbase-0.98.19/lib/
export HBASE_IDENT_STRING=my
export HBASE_PID_DIR=${HBASE_HOME}/tmp
export HBASE_MANAGES_ZK=true
```

##### 配置HBASE_HOME/conf/hbase-site.xml，hbase-default.xml并不在conf目录, 需要从./src/main/resources/目录拷贝并且改名为hbase-site.xml

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <!-- 使用本地文件系统 -->
    <value>file:///software/hbase-0.98.19/hbase_dir</value>
    <!-- 使用本地的HDFS文件系统 -->
    <!-- <value>hdfs://Ben:9000/hbase</value> -->
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/software/hbase-0.98.19/zookeeper_dir</value>
  </property>
</configuration>
```


##### 控制台终端

```
# 构建镜像
$ docker build -t="birdben/hbase:v1" .
# 执行已经构件好的镜像，挂载在宿主机器的存储路径也不同，-h设置hostname，Hadoop配置文件需要使用
$ docker run -p 9999:22 -p 60010:60010 -t -i 'birdben/hbase:v1'
```

##### supervisor无法监控hbase

```
supervisor监控hbase不成功，是因为supervisor启动的程序必须是非daemon的启动方式，之前的文章已经反复提过好几次这个问题，但是${HBASE_HOME}/bin/start-hbase.sh的启动脚本，实际上是调用了hbase-daemon.sh的daemon启动脚本，所以这里supervisor启动hbase后，hbase进程的状态仍然是hbase (exit status 0; expected)，是因为无法监控hbase进程，但是不影响hbase的正常启动和使用
```

##### 使用hbase shell登录HBase服务器进行操作

```
# 通过ssh远程连接使用admin账号远程登录HBase的Docker容器
$ cd /software/hbase-0.98.19/bin/
$ ./hbase shell
2016-06-30 06:03:57,427 INFO  [main] Configuration.deprecation: hadoop.native.lib is deprecated. Instead, use io.native.lib.available
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.98.19-hadoop2, r1e527e73bc539a04ba0fa4ed3c0a82c7e9dd7d15, Fri Apr 22 19:07:24 PDT 2016
hbase(main):001:0>

# 查询所有的Table
hbase(main):001:0> list

# 创建新表
hbase(main):001:0> create 'test','name'
0 row(s) in 0.5780 seconds
=> Hbase::Table - test

# 再次查询所有的Table
hbase(main):001:0> list
TABLE
test
1 row(s) in 0.0100 seconds
=> ["test"]
```

##### 浏览器查看

```
# 查看HBase：
http://10.211.55.4:60010/master-status
```

##### 启动hbase可能会报错 Address already in use
```
master: java.net.BindException: Address already in use
master:         at sun.nio.ch.Net.bind(Native Method)
master:         at sun.nio.ch.ServerSocketChannelImpl.bind(ServerSocketChannelImpl.java:124)
master:         at sun.nio.ch.ServerSocketAdaptor.bind(ServerSocketAdaptor.java:59)
master:         at sun.nio.ch.ServerSocketAdaptor.bind(ServerSocketAdaptor.java:52)
master:         at org.apache.zookeeper.server.NIOServerCnxnFactory.configure(NIOServerCnxnFactory.java:111)
master:         at org.apache.zookeeper.server.quorum.QuorumPeerMain.runFromConfig(QuorumPeerMain.java:130)
master:         at org.apache.hadoop.hbase.zookeeper.HQuorumPeer.runZKServer(HQuorumPeer.java:73)
master:         at org.apache.hadoop.hbase.zookeeper.HQuorumPeer.main(HQuorumPeer.java:63)

# 报错的原因是Zookeeper的端口被占用了或者Zookeeper已经启动了。
export HBASE_MANAGES_ZK=true
# 这个参数表示启动hbase之前自动启动zk
# 解决的办法有2种：
1.启动hbase的之前kill掉所有的zk进程，让hbase启动zk
2.将参数HBASE_MANAGES_ZK 改成false
# 在hbase之前手动启动zk

# 我这里使用的是方式一，可能之前zk的2181端口被其他应用占用了，所以kill掉所有占用2181端口的进程，再启动hbase就好用了
```


参考文章：

- http://hbase.apache.org/book.html#quickstart
- http://os.51cto.com/art/201211/363518.htm
- http://blog.csdn.net/21aspnet/article/details/18776833
- http://blog.chinaunix.net/uid-77311-id-4580084.html
- http://blog.csdn.net/lxpbs8851/article/details/8268894
- http://my.oschina.net/u/914897/blog/526014