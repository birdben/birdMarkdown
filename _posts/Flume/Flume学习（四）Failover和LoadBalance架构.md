---
title: "Flume学习（四）Failover架构"
date: 2016-08-25 13:18:47
tags: [Flume]
categories: [Log]
---

##### Failover架构

![Failover](http://img.blog.csdn.net/20160824170052276?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

我这里是用本机模拟此架构，AgentX是采集端，分别写入Sink1和Sink2，Agent1和Agent2是Collect端。此架构允许Agent1和Agent2部分停机，需要在采集层（AgentX）每一个Sink同时指向Collect层的2个相同的Flume Agent（Agent1和Agent2）。所以使用failover架构就是为了防止Collect层Flume Agent（Agent1和Agent2）因为故障或例行停机维护。

##### AgentX节点的flume\_failover.conf配置

```
agentX.sources = sX
agentX.channels = chX
agentX.sinks = sk1 sk2

agentX.sources.sX.channels = chX
agentX.sources.sX.type = exec
agentX.sources.sX.command = tail -F /Users/yunyu/Downloads/command.log

agentX.channels.chX.type = memory
agentX.channels.chX.capacity = 1000
agentX.channels.chX.transactionCapacity = 100

# 配置sinks，这里我们指定了2个相同的Agent（即Agent1和Agent2，这里我们使用本机测试，所以是两个相同的Agent进程，hostname都是本机IP，只是port不同用于区分）
agentX.sinks.sk1.channel = chX
agentX.sinks.sk1.type = avro
agentX.sinks.sk1.hostname = 10.10.1.23
agentX.sinks.sk1.port = 44441

agentX.sinks.sk2.channel = chX
agentX.sinks.sk2.type = avro
agentX.sinks.sk2.hostname = 10.10.1.23
agentX.sinks.sk2.port = 44442

# 配置failover组信息，把上面的两个sink配置成一个group，并且指定类型为failover
agentX.sinkgroups = g1
agentX.sinkgroups.g1.sinks = sk1 sk2
agentX.sinkgroups.g1.processor.type = failover
agentX.sinkgroups.g1.processor.priority.sk1 = 9
agentX.sinkgroups.g1.processor.priority.sk2 = 7
agentX.sinkgroups.g1.processor.maxpenalty = 10000
```

##### Agent1节点的flume\_failover\_agent1.conf配置

```
agent1.sources = s1
agent1.channels = ch1
agent1.sinks = sk1

agent1.sources.s1.channels = ch1
agent1.sources.s1.type = avro
agent1.sources.s1.bind = 10.10.1.23
agent1.sources.s1.port = 44441

agent1.channels.ch1.type = memory
agent1.channels.ch1.capacity = 1000
agent1.channels.ch1.transactionCapacity = 100

agent1.sinks.sk1.channel = ch1
agent1.sinks.sk1.type = logger
```


##### Agent1节点的flume\_failover\_agent2.conf配置

```
agent2.sources = s2
agent2.channels = ch2
agent2.sinks = sk2

agent2.sources.s2.channels = ch2
agent2.sources.s2.type = avro
agent2.sources.s2.bind = 10.10.1.23
agent2.sources.s2.port = 44442

agent2.channels.ch2.type = memory
agent2.channels.ch2.capacity = 1000
agent2.channels.ch2.transactionCapacity = 100

agent2.sinks.sk2.channel = ch2
agent2.sinks.sk2.type = logger
```

注意：

- 如果是多台机器实验，Agent1和Agent2的flume.conf配置其实可以是一样的，只是我这里使用的本机测试，所以需要指定不同的port来模拟2台不同的机器，flume.conf配置文件也分开了。

##### 启动Flume Agent

```
# 启动采集端，AgentX
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_failover.conf -Dflume.root.logger=DEBUG,console -n agentX

# 启动2个Collect端，Agent1和Agent2
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_failover_agent1.conf -Dflume.root.logger=DEBUG,console -n agent1
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume_failover_agent2.conf -Dflume.root.logger=DEBUG,console -n agent2
```

这时候我们发现采集的日志都在Agent1中的控制台输出，Agent2并没有日志输出。但是我们查看44441和44442端口号发现AgentX和Agent1，Agent2都保持TCP连接的。

```
$ ps -ef | grep flume
  501  8446  6602   0  5:45PM ttys000    0:01.02 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_2/conf:/Users/yunyu/dev/flume-1.6.0_agent_2/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_failover_agent2.conf -n agent2
  501  8455  4754   0  5:45PM ttys001    0:01.21 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_failover.conf -n agentX
  501  8436  5574   0  5:45PM ttys003    0:01.28 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_1/conf:/Users/yunyu/dev/flume-1.6.0_agent_1/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_failover_agent1.conf -n agent1
  501  8466  5608   0  5:45PM ttys005    0:00.00 grep flume

$ lsof -i:44441
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    8436 yunyu  156u  IPv6 0xd0e0ada8bdad6435      0t0  TCP localhost:44441 (LISTEN)
java    8436 yunyu  161u  IPv6 0xd0e0ada8c2f7d3d5      0t0  TCP localhost:44441->localhost:52471 (ESTABLISHED)
java    8455 yunyu  210u  IPv6 0xd0e0ada8c3175955      0t0  TCP localhost:52471->localhost:44441 (ESTABLISHED)

$ lsof -i:44442
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    8446 yunyu  156u  IPv6 0xd0e0ada8e3f0b3f5      0t0  TCP localhost:44442 (LISTEN)
java    8446 yunyu  161u  IPv6 0xd0e0ada8bdad43f5      0t0  TCP localhost:44442->localhost:52470 (ESTABLISHED)
java    8455 yunyu  156u  IPv6 0xd0e0ada8c3152955      0t0  TCP localhost:52470->localhost:44442 (ESTABLISHED)
```

此时我们杀掉Agent1的进程，我们会看到AgentX的控制台会报错提示：Connection Refused无法连接到Agent1。

```
# flume Agent1的进程已经没有了
$ ps -ef | grep flume
  501  8446  6602   0  5:45PM ttys000    0:05.13 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0_agent_2/conf:/Users/yunyu/dev/flume-1.6.0_agent_2/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_failover_agent2.conf -n agent2
  501  8455  4754   0  5:45PM ttys001    0:07.85 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume_failover.conf -n agentX
  501  8732  5608   0  6:08PM ttys005    0:00.00 grep flume

# 再次查看44441端口发现TCP连接已经断开了
$ lsof -i:44441
```

这个时候我们再有日志采集会发现日志都输出到Agent2的控制台了，说明我们的failover机制生效了。


参考文章：

- http://shiyanjun.cn/archives/1497.html
- http://tech.meituan.com/mt-log-system-arch.html