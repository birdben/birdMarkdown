---
title: "Flume学习（三）Flume多个Agent实例"
date: 2016-08-24 21:48:19
tags: [Flume]
categories: [Log]
---

##### 多个Agent的数据汇聚到同一个Agent

![多个Agent的数据汇聚到同一个Agent](http://7xnrdo.com1.z0.glb.clouddn.com/2014/flume-join-agent.png)

我这里是用本机模拟此架构，三个日志收集Flume Agent节点和一个日志Flume Collector节点

##### Agent1节点的flume.conf配置

```
agent1.sources = system-logfile-source
agent1.channels = ch1
agent1.sinks = flume-avro-sink

# 这里收集的是/var/log/system.log日志文件
agent1.sources.system-logfile-source.channels = ch1
agent1.sources.system-logfile-source.type = exec
agent1.sources.system-logfile-source.command = tail -F /var/log/system.log

agent1.channels.ch1.type = memory
agent1.channels.ch1.capacity = 1000
agent1.channels.ch1.transactionCapacity = 100

# Agent1设置sink的hostname是10.10.1.23（我本机的IP地址），也就是该Agent要向10.10.1.23主机发送数据
agent1.sinks.flume-avro-sink.channel = ch1
agent1.sinks.flume-avro-sink.type = avro
agent1.sinks.flume-avro-sink.hostname = 10.10.1.23
agent1.sinks.flume-avro-sink.port = 41414
```

##### Agent2节点的flume.conf配置

```
agent2.sources = install-logfile-source
agent2.channels = ch2
agent2.sinks = flume-avro-sink

# 这里收集的是/var/log/install.log日志文件
agent2.sources.install-logfile-source.channels = ch2
agent2.sources.install-logfile-source.type = exec
agent2.sources.install-logfile-source.command = tail -F /var/log/install.log

agent2.channels.ch2.type = memory
agent2.channels.ch2.capacity = 1000
agent2.channels.ch2.transactionCapacity = 100

# Agent2设置sink的hostname是10.10.1.23（我本机的IP地址），也就是该Agent要向10.10.1.23主机发送数据
agent2.sinks.flume-avro-sink.channel = ch2
agent2.sinks.flume-avro-sink.type = avro
agent2.sinks.flume-avro-sink.hostname = 10.10.1.23
agent2.sinks.flume-avro-sink.port = 41414
```

##### Agent3节点的flume.conf配置

```
agent3.sources = command-logfile-source
agent3.channels = ch3
agent3.sinks = flume-avro-sink

# 这里收集的是/Users/yunyu/Downloads/command.log日志文件，这个日志文件是我自己定义的（请根据自己的实际环境配置相应的log日志文件）
agent3.sources.command-logfile-source.channels = ch3
agent3.sources.command-logfile-source.type = exec
agent3.sources.command-logfile-source.command = tail -F /Users/yunyu/Downloads/command.log

agent3.channels.ch3.type = memory
agent3.channels.ch3.capacity = 1000
agent3.channels.ch3.transactionCapacity = 100

# Agent3设置sink的hostname是10.10.1.23（我本机的IP地址），也就是该Agent要向10.10.1.23主机发送数据
agent3.sinks.flume-avro-sink.channel = ch3
agent3.sinks.flume-avro-sink.type = avro
agent3.sinks.flume-avro-sink.hostname = 10.10.1.23
agent3.sinks.flume-avro-sink.port = 41414
```

##### Collector节点的flume_collect.conf配置

```
agentX.sources = flume-avro-sink
agentX.channels = chX
agentX.sinks = flume-collect-sink

# 监听的IP地址是10.10.1.23。三个Agent节点的sinks的传输协议类型要和Collector节点的sources的传输协议类型一致，这里传输协议都是avro。
agentX.sources.flume-avro-sink.channels = chX
agentX.sources.flume-avro-sink.type = avro
agentX.sources.flume-avro-sink.bind = 10.10.1.23
agentX.sources.flume-avro-sink.port = 41414
agentX.sources.flume-avro-sink.threads = 8

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

# 这里是将接收到的数据，以文件的形式存储起来，保存路径是/Users/yunyu/Downloads/sinkout/
agentX.sinks.flume-collect-sink.channel = chX
agentX.sinks.flume-collect-sink.type = file_roll
agentX.sinks.flume-collect-sink.batchSize = 100
agentX.sinks.flume-collect-sink.serializer = TEXT
agentX.sinks.flume-collect-sink.sink.directory = /Users/yunyu/Downloads/sinkout/
```

##### 注意
这里需要注意一下sources和sinks的配置，我们在三个Agent节点都指定了sinks的hostname=10.10.1.23，但是Collector节点指定的sources的bind=10.10.1.23，这两个参数需要注意下，我开始的时候就配置错了，在sinks使用的bind=10.10.1.23，而没有使用hostname参数，Flume启动的时候就会提示"java.lang.IllegalStateException: No hostname specified"这个错误，后来查了一下官网的配置，发现是我自己把sources和sinks的绑定主机的参数搞混了

- sources使用的是bind（意思是监听主机）
- sinks使用的是hostname（意思是传输数据的主机）

##### 分别启动Collector和三个Agent节点

```
# 最好先启动Collector
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_collect.conf -Dflume.root.logger=DEBUG,console -n agentX
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume.conf -Dflume.root.logger=DEBUG,console -n agent1
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume.conf -Dflume.root.logger=DEBUG,console -n agent2
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume.conf -Dflume.root.logger=DEBUG,console -n agent3
```

##### 启动三个Agent节点分别会看到如下输出信息

```
2016-08-24 15:35:31,144 (New I/O server boss #1 ([id: 0x0c4ee010, /10.10.1.23:41414])) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xad3b4304, /10.10.1.23:50791 => /10.10.1.23:41414] OPEN
2016-08-24 15:35:31,146 (New I/O  worker #1) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xad3b4304, /10.10.1.23:50791 => /10.10.1.23:41414] BOUND: /10.10.1.23:41414
2016-08-24 15:35:31,146 (New I/O  worker #1) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xad3b4304, /10.10.1.23:50791 => /10.10.1.23:41414] CONNECTED: /10.10.1.23:50791
```


```
2016-08-24 15:36:03,623 (New I/O server boss #1 ([id: 0x0c4ee010, /10.10.1.23:41414])) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xdda0a0fd, /10.10.1.23:50797 => /10.10.1.23:41414] OPEN
2016-08-24 15:36:03,623 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xdda0a0fd, /10.10.1.23:50797 => /10.10.1.23:41414] BOUND: /10.10.1.23:41414
2016-08-24 15:36:03,623 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0xdda0a0fd, /10.10.1.23:50797 => /10.10.1.23:41414] CONNECTED: /10.10.1.23:50797
```

```
2016-08-24 15:38:27,270 (New I/O server boss #1 ([id: 0x0c4ee010, /10.10.1.23:41414])) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x909310e6, /10.10.1.23:50822 => /10.10.1.23:41414] OPEN
2016-08-24 15:38:27,270 (New I/O  worker #3) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x909310e6, /10.10.1.23:50822 => /10.10.1.23:41414] BOUND: /10.10.1.23:41414
2016-08-24 15:38:27,271 (New I/O  worker #3) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x909310e6, /10.10.1.23:50822 => /10.10.1.23:41414] CONNECTED: /10.10.1.23:50822
```
我们会看到每个Agent实际上是启动了一个NettyServer进行通信，三个Agent的启动log都会在本机IP：10.10.1.23上开启一个端口号与Collector的端口号41414进行通信

```
10.10.1.23:50791 => /10.10.1.23:41414
10.10.1.23:50797 => /10.10.1.23:41414
10.10.1.23:50822 => /10.10.1.23:41414
```

我们在分别查询一下当前flume的所有进程和上面对应的三个端口号，会发现50791, 50797, 50822这三个端口号正如上面所说的是Agent1 -> AgentX, Agent2 -> AgentX, Agent3 -> AgentX的通信端口

```
$ ps -ef | grep flume
  501  7824  6602   0  3:36PM ttys000    0:02.20 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_2/conf:/Users/yunyu/dev/flume-1.6.0_agent_2/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume.conf -n agent2
  501  7804  4754   0  3:35PM ttys001    0:01.96 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_collect.conf -n agentX
  501  7921  7903   0  3:39PM ttys002    0:00.00 grep flume
  501  7814  5574   0  3:35PM ttys003    0:02.57 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_1/conf:/Users/yunyu/dev/flume-1.6.0_agent_1/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume.conf -n agent1
  501  7886  5608   0  3:38PM ttys005    0:01.58 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_3/conf:/Users/yunyu/dev/flume-1.6.0_agent_3/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume.conf -n agent3
```

```
$ lsof -i:50791
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    7804 yunyu  162u  IPv6 0xd0e0ada8bdad6435      0t0  TCP localhost:41414->localhost:50791 (ESTABLISHED)
java    7814 yunyu  156u  IPv6 0xd0e0ada8c2f7e955      0t0  TCP localhost:50791->localhost:41414 (ESTABLISHED)

$ lsof -i:50797
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    7804 yunyu  164u  IPv6 0xd0e0ada8bdad43f5      0t0  TCP localhost:41414->localhost:50797 (ESTABLISHED)
java    7824 yunyu  156u  IPv6 0xd0e0ada8c3152955      0t0  TCP localhost:50797->localhost:41414 (ESTABLISHED)

$ lsof -i:50822
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    7804 yunyu  165u  IPv6 0xd0e0ada8c3175955      0t0  TCP localhost:41414->localhost:50822 (ESTABLISHED)
java    7886 yunyu  156u  IPv6 0xd0e0ada8c3175eb5      0t0  TCP localhost:50822->localhost:41414 (ESTABLISHED)
```

##### 验证结果

这时候我们只要在system.log, install.log, command.log中产生任何日志，都会输出对应的日志文件到/Users/yunyu/Downloads/sinkout/路径

参考文章：

- http://shiyanjun.cn/archives/915.html
- http://blog.javachen.com/2014/07/22/flume-ng.html
- http://flume.apache.org/FlumeUserGuide.html