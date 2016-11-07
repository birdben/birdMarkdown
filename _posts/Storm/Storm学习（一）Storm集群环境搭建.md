---
title: "Storm学习（一）Storm集群环境搭建"
date: 2016-11-07 14:57:13
tags: [Storm]
categories: [Storm]
---

今天开始搭建Storm集群环境了，主要参考官网的步骤一点一点学习的

搭建Storm集群的主要步骤如下:

- 搭建好一个Zookeeper集群
- 安装好依赖的环境
- 下载并解压一个Storm版本
- 补充主要的配置到storm.yaml配置文件
- 后台执行Storm启动脚本

### Zookeeper集群环境搭建

这里我们不做详细介绍了，Zookeeper集群环境搭建我之前单独写过一篇文章，请参考。

### 安装依赖环境

Storm依赖于Java环境和Python环境，具体版本如下：

```
Java 7
Python 2.6.6
```

可能Storm在不同的Java和Python版本下会不好用

#### 安装JDK

省略

#### 安装Python

```
# 下载Python2.6.6
$ wget http://www.python.org/ftp/python/2.6.6/Python-2.6.6.tar.bz2

# 编译安装Python2.6.6
$ tar –jxvf Python-2.6.6.tar.bz2
$ cd Python-2.6.6
$ ./configure
$ make
$ make install

# 测试Python
$ python -V
Python 2.6.6
```

### 下载Storm

在Storm的GitHub中选择一个Storm版本下载安装，这里我选择的0.10.2版本

- http://storm.apache.org/downloads.html

```
# 下载Storm安装包
$ curl -O http://apache.fayea.com/storm/apache-storm-0.10.2/apache-storm-0.10.2.tar.gz

# 解压Storm压缩包
$ tar -xvf apache-storm-0.10.2.tar
```

### 修改storm.yaml配置文件

Storm启动会加载conf/storm.yaml配置文件，该配置文件会覆盖掉defaults.yaml配置文件中的配置。下面介绍少部分重要的配置

- storm.zookeeper.servers : Storm依赖的Zookeeper集群地址

```
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

如果Zookeeper集群使用的不是默认的端口号，可以通过storm.zookeeper.port配置来修改，否则会出现通信错误。

- storm.local.dir : Nimbus和Supervisor进程用于存储少量状态，如jars、confs等的本地磁盘目录，需要提前创建该目录并给以足够的访问权限。需要在每个Storm机器中都创建这样一个目录。

```
storm.local.dir: "/mnt/storm"
```

也可以使用相对路径，相对于$STORM_HOME（即Storm的安装路径），如果不设置该配置就是用默认目录$STORM_HOME/storm-local

```
storm.local.dir: $STORM_HOME/storm-local
```

- nimbus.seeds : Storm集群Nimbus机器地址，各个Supervisor工作节点需要知道哪个机器是Nimbus，以便下载Topologies的jars、confs等文件。

```
nimbus.seeds: ["111.222.333.44"]
```

这里推荐使用机器的hostname，如果你想设置Nimbus的HA高可用，你必须设置每个正在运行的Nimbus的hostname。如果是假的分布式集群，可以使用默认值，但是建议设置Nimbus的hostname。

- supervisor.slots.ports : 对于每个Supervisor工作节点，需要配置该工作节点可以运行的worker数量。每个worker占用一个单独的端口用于接收消息，该配置选项即用于定义哪些端口是可被worker使用的。默认情况下，每个节点上可运行4个workers，分别在6700、6701、6702和6703端口

```
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

### 健康检查（下面这段翻译的不够准确，仅供参考）

Storm提供一种机制 —— 管理者可以配置一个监控者周期的执行管理者提供的脚本，检查node节点是否健康。管理者有决定node节点在健康状态。如果一个脚本查明node节点处于非健康状态，必须打印以ERROR开头标准化输出错误。监控者周期的执行脚本在storm.health.check.dir并且检查输出结果。如果脚本的输出结果包括ERROR，监控者将会关闭worker。

如果监控者正在运行与监控"/bin/storm node-health-check"可以被调用，以确定监控者是否被启动，或者该node节点是否是不健康的

```
# 健康检查的目录配置如下
storm.health.check.dir: "healthchecks"
# 健康检查等待超时时间配置
storm.health.check.timeout.ms: 5000
```

### 配置外部依赖库和环境变量（可选）

如果你需要支持外部依赖库或者自定义插件，你可以定位这些jar到extlib和extlib-daemon目录下。注意extlib-daemon目录存储的jar只被用于daemons启动(Nimbus, Supervisor, DRPC, UI, Logviewer)，如，HDFS和自定义的定时任务库。因此STORM_EXT_CLASSPATH和STORM_EXT_CLASSPATH_DAEMON这两个环境变量都需要被配置，为了外部依赖classpath和后台启动的外部依赖classpath。

### 启动Storm后台进程

最后一步就是启动Storm的后台进程，很关键的一点这些后台进程都在监控下。Storm是快速失败（fail-fast)，意味着无论什么时候发生错误，进程都会停止。Storm被设计成这样可以在任何点安全的停止，并且当进程重启时可以正确的被恢复。这也是说明Storm为什么是无状态的系统。如果Nimbus或者Supervisors重启，正在运行的topologies不会受到影响。

下面是如何启动Storm的后台进程

```
Nimbus: Run the command "bin/storm nimbus" under supervision on the master machine.

Supervisor: Run the command "bin/storm supervisor" under supervision on each worker machine. The supervisor daemon is responsible for starting and stopping worker processes on that machine.

UI: Run the Storm UI (a site you can access from the browser that gives diagnostics on the cluster and topologies) by running the command "bin/storm ui" under supervision. The UI can be accessed by navigating your web browser to http://{ui host}:8080.
```

Storm后台进程被启动后，将在Storm安装部署目录下的logs/子目录下生成各个进程的日志文件

以上是翻译自Storm的官网，接下来是针对于我自己的Storm集群的配置过程

### Storm集群搭建步骤

修改storm.yaml配置如下

```
storm.zookeeper.servers:    - "hadoop2"    - "hadoop3"nimbus.host: "hadoop1"
```

注意：这里hadoop1, hadoop2, hadoop3是我之前安装Hadoop集群环境对应的三台机器的hostname，需要根据个人实际环境进行修改。这里我将hadoop1这台机器作为Nimbus，hadoop2和hadoop3这两台机器作为Supervisor。其他配置都使用默认的配置。

启动Nimbus

```
$ bin/storm nimbusRunning: /usr/local/java/bin/java -server -Ddaemon.name=nimbus -Dstorm.options= -Dstorm.home=/data/storm-0.10.2 -Dstorm.log.dir=/data/storm-0.10.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /data/storm-0.10.2/lib/storm-core-0.10.2.jar:/data/storm-0.10.2/lib/slf4j-api-1.7.7.jar:/data/storm-0.10.2/lib/clojure-1.6.0.jar:/data/storm-0.10.2/lib/disruptor-2.10.4.jar:/data/storm-0.10.2/lib/servlet-api-2.5.jar:/data/storm-0.10.2/lib/log4j-api-2.1.jar:/data/storm-0.10.2/lib/log4j-core-2.1.jar:/data/storm-0.10.2/lib/minlog-1.2.jar:/data/storm-0.10.2/lib/reflectasm-1.07-shaded.jar:/data/storm-0.10.2/lib/log4j-over-slf4j-1.6.6.jar:/data/storm-0.10.2/lib/asm-4.0.jar:/data/storm-0.10.2/lib/hadoop-auth-2.4.0.jar:/data/storm-0.10.2/lib/kryo-2.21.jar:/data/storm-0.10.2/lib/log4j-slf4j-impl-2.1.jar:/data/storm-0.10.2/conf -Xmx1024m -Dlogfile.name=nimbus.log -Dlog4j.configurationFile=/data/storm-0.10.2/log4j2/cluster.xml backtype.storm.daemon.nimbus
```

启动Supervisor

```
$ bin/storm supervisorRunning: /usr/local/java/bin/java -server -Ddaemon.name=supervisor -Dstorm.options= -Dstorm.home=/data/storm-0.10.2 -Dstorm.log.dir=/data/storm-0.10.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /data/storm-0.10.2/lib/asm-4.0.jar:/data/storm-0.10.2/lib/servlet-api-2.5.jar:/data/storm-0.10.2/lib/slf4j-api-1.7.7.jar:/data/storm-0.10.2/lib/reflectasm-1.07-shaded.jar:/data/storm-0.10.2/lib/clojure-1.6.0.jar:/data/storm-0.10.2/lib/hadoop-auth-2.4.0.jar:/data/storm-0.10.2/lib/log4j-api-2.1.jar:/data/storm-0.10.2/lib/log4j-slf4j-impl-2.1.jar:/data/storm-0.10.2/lib/storm-core-0.10.2.jar:/data/storm-0.10.2/lib/log4j-over-slf4j-1.6.6.jar:/data/storm-0.10.2/lib/kryo-2.21.jar:/data/storm-0.10.2/lib/minlog-1.2.jar:/data/storm-0.10.2/lib/log4j-core-2.1.jar:/data/storm-0.10.2/lib/disruptor-2.10.4.jar:/data/storm-0.10.2/conf -Xmx256m -Dlogfile.name=supervisor.log -Dlog4j.configurationFile=/data/storm-0.10.2/log4j2/cluster.xml backtype.storm.daemon.supervisor
```

启动UI

```
$ bin/storm uiRunning: /usr/local/java/bin/java -server -Ddaemon.name=ui -Dstorm.options= -Dstorm.home=/data/storm-0.10.2 -Dstorm.log.dir=/data/storm-0.10.2/logs -Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib -Dstorm.conf.file= -cp /data/storm-0.10.2/lib/storm-core-0.10.2.jar:/data/storm-0.10.2/lib/slf4j-api-1.7.7.jar:/data/storm-0.10.2/lib/clojure-1.6.0.jar:/data/storm-0.10.2/lib/disruptor-2.10.4.jar:/data/storm-0.10.2/lib/servlet-api-2.5.jar:/data/storm-0.10.2/lib/log4j-api-2.1.jar:/data/storm-0.10.2/lib/log4j-core-2.1.jar:/data/storm-0.10.2/lib/minlog-1.2.jar:/data/storm-0.10.2/lib/reflectasm-1.07-shaded.jar:/data/storm-0.10.2/lib/log4j-over-slf4j-1.6.6.jar:/data/storm-0.10.2/lib/asm-4.0.jar:/data/storm-0.10.2/lib/hadoop-auth-2.4.0.jar:/data/storm-0.10.2/lib/kryo-2.21.jar:/data/storm-0.10.2/lib/log4j-slf4j-impl-2.1.jar:/data/storm-0.10.2:/data/storm-0.10.2/conf -Xmx768m -Dlogfile.name=ui.log -Dlog4j.configurationFile=/data/storm-0.10.2/log4j2/cluster.xml backtype.storm.ui.core
```

启动成功之后访问http://hadoop1:8080，效果图如下

![Storm UI](http://img.blog.csdn.net/20161107192753473?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


参考文章：

- http://storm.apache.org/releases/0.10.2/Setting-up-a-Storm-cluster.html