---
title: "Flume学习（一）Flume环境搭建"
date: 2016-08-23 14:21:09
tags: [Flume]
categories: [Log]
---

#### Flume安装
```
$ wget http://apache.fayea.com/flume/1.6.0/apache-flume-1.6.0-bin.tar.gz
$ tar -xvf apache-flume-1.6.0-bin.tar
$ mv apache-flume-1.6.0-bin flume-1.6.0
$ cd flume-1.6.0

# 修改flume配置文件
$ cp conf/flume-conf.properties.template conf/flume.conf
$ cp conf/flume-env.sh.template conf/flume-env.sh
```

#### Flume组件介绍

- Event：一个数据单元，带有一个可选的消息头
- Flow：Event从源点到达目的点的迁移的抽象
- Client：操作位于源点处的Event，将其发送到Flume Agent（下面介绍的AvroClient用法）
- Agent：一个独立的Flume进程，包含组件Source、Channel、Sink
- Source：用来消费传递到该组件的Event（从数据源获取生成event data）
- Channel：中转Event的一个临时存储，保存有Source组件传递过来的Event（接收source给put来的event data）
- Sink：从Channel中读取并移除Event（从channel取走event data），将Event传递到Flow Pipeline中的下一个Agent（如果有的话）

![Flume组件图](http://flume.apache.org/_images/UserGuide_image00.png)

- 参考下面的图比较容易理解AvroClient和Flume Agent的关系

![Flume Client](https://blogs.apache.org/flume/mediaresource/ab0d50f6-a960-42cc-971e-3da38ba3adad)

#### Flume配置文件格式
```
# list the sources, sinks and channels for the agent
<Agent>.sources = <Source1> <Source2>
<Agent>.sinks = <Sink1> <Sink2>
<Agent>.channels = <Channel1> <Channel2>

# set channel for source
<Agent>.sources.<Source1>.channels = <Channel1> <Channel2>
<Agent>.sources.<Source2>.channels = <Channel1> <Channel2>

# set channel
<Agent>.channels.<Channel1>.XXX = XXX
<Agent>.channels.<Channel2>.XXX = XXX

# set channel for sink
<Agent>.sinks.<Sink1>.channel = <Channel1>
<Agent>.sinks.<Sink2>.channel = <Channel2>
```

#### Flume配置文件
```
# 这里对比上面的Flume组件图来看此配置信息就比较容易理解了
# 定义当前Agent的所有的组件（sources,channels,sinks），Flume这里的组件和Logstash的input，filter，output非常类似
agent1.sources = avro-source1
agent1.channels = ch1
agent1.sinks = log-sink1

# 定义sources组件的具体配置
# 设置source的目标是哪个channel
agent1.sources.avro-source1.channels = ch1
# 设置source的输入信息类型，表示该source接收的数据协议为avro（也就是说resource要通过avro-cliet向其发送数据）
agent1.sources.avro-source1.type = avro
# 设置source的监听主机的IP地址，或者hostname
# 这里我使用的是本机IP：10.10.1.23，如果是本机测试也可以使用localhost
agent1.sources.avro-source1.bind = 10.10.1.23
# 设置source的监听主机的port
agent1.sources.avro-source1.port = 41414

# 定义channels组件的具体配置
# 设置Channel的类型
agent1.channels.ch1.type = memory

# 定义sinks组件的具体配置
# 设置sink的来源于哪个channel
agent1.sinks.log-sink1.channel = ch1
# 设置sink的输出信息类型，将数据输出至Flume的日志中(也就是打印在屏幕上)
agent1.sinks.log-sink1.type = logger
```

Avro协议参考：

- http://langyu.iteye.com/blog/708568

#### 启动Flume Agent端
```
# 这里指定的agent1名称必须和flume.conf配置文件中的agent名称一致
# 这里的启动命令是./bin/flume-ng agent，开始监听41414端口
$ ./bin/flume-ng agent --conf ./conf/ -f conf/flume.conf -Dflume.root.logger=DEBUG,console -n agent1

+ exec /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp '/Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/*' -Djava.library.path= org.apache.flume.node.Application -f conf/flume.conf -n agent1
2016-08-24 10:15:05,547 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.node.PollingPropertiesFileConfigurationProvider.start(PollingPropertiesFileConfigurationProvider.java:61)] Configuration provider starting
2016-08-24 10:15:05,551 (lifecycleSupervisor-1-0) [DEBUG - org.apache.flume.node.PollingPropertiesFileConfigurationProvider.start(PollingPropertiesFileConfigurationProvider.java:78)] Configuration provider started
2016-08-24 10:15:05,553 (conf-file-poller-0) [DEBUG - org.apache.flume.node.PollingPropertiesFileConfigurationProvider$FileWatcherRunnable.run(PollingPropertiesFileConfigurationProvider.java:126)] Checking file:conf/flume.conf for changes
2016-08-24 10:15:05,555 (conf-file-poller-0) [INFO - org.apache.flume.node.PollingPropertiesFileConfigurationProvider$FileWatcherRunnable.run(PollingPropertiesFileConfigurationProvider.java:133)] Reloading configuration file:conf/flume.conf
2016-08-24 10:15:05,559 (conf-file-poller-0) [INFO - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.addProperty(FlumeConfiguration.java:1017)] Processing:log-sink1
2016-08-24 10:15:05,559 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.addProperty(FlumeConfiguration.java:1021)] Created context for log-sink1: type
2016-08-24 10:15:05,560 (conf-file-poller-0) [INFO - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.addProperty(FlumeConfiguration.java:1017)] Processing:log-sink1
2016-08-24 10:15:05,560 (conf-file-poller-0) [INFO - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.addProperty(FlumeConfiguration.java:931)] Added sinks: log-sink1 Agent: agent1
2016-08-24 10:15:05,563 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.isValid(FlumeConfiguration.java:314)] Starting validation of configuration for agent: agent1, initial-configuration: AgentConfiguration[agent1]
SOURCES: {avro-source1={ parameters:{port=41414, channels=ch1, type=avro, bind=10.10.1.23} }}
CHANNELS: {ch1={ parameters:{type=memory} }}
SINKS: {log-sink1={ parameters:{type=logger, channel=ch1} }}

2016-08-24 10:15:05,566 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.validateChannels(FlumeConfiguration.java:469)] Created channel ch1
2016-08-24 10:15:05,573 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.validateSinks(FlumeConfiguration.java:675)] Creating sink: log-sink1 using LOGGER
2016-08-24 10:15:05,574 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration$AgentConfiguration.isValid(FlumeConfiguration.java:372)] Post validation configuration for agent1
AgentConfiguration created without Configuration stubs for which only basic syntactical validation was performed[agent1]
SOURCES: {avro-source1={ parameters:{port=41414, channels=ch1, type=avro, bind=10.10.1.23} }}
CHANNELS: {ch1={ parameters:{type=memory} }}
AgentConfiguration created with Configuration stubs for which full validation was performed[agent1]
SINKS: {log-sink1=ComponentConfiguration[log-sink1]
  CONFIG:
    CHANNEL:ch1
}

2016-08-24 10:15:05,574 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration.validateConfiguration(FlumeConfiguration.java:136)] Channels:ch1

2016-08-24 10:15:05,575 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration.validateConfiguration(FlumeConfiguration.java:137)] Sinks log-sink1

2016-08-24 10:15:05,575 (conf-file-poller-0) [DEBUG - org.apache.flume.conf.FlumeConfiguration.validateConfiguration(FlumeConfiguration.java:138)] Sources avro-source1

2016-08-24 10:15:05,575 (conf-file-poller-0) [INFO - org.apache.flume.conf.FlumeConfiguration.validateConfiguration(FlumeConfiguration.java:141)] Post-validation flume configuration contains configuration for agents: [agent1]
2016-08-24 10:15:05,577 (conf-file-poller-0) [INFO - org.apache.flume.node.AbstractConfigurationProvider.loadChannels(AbstractConfigurationProvider.java:145)] Creating channels
2016-08-24 10:15:05,584 (conf-file-poller-0) [INFO - org.apache.flume.channel.DefaultChannelFactory.create(DefaultChannelFactory.java:42)] Creating instance of channel ch1 type memory
2016-08-24 10:15:05,591 (conf-file-poller-0) [INFO - org.apache.flume.node.AbstractConfigurationProvider.loadChannels(AbstractConfigurationProvider.java:200)] Created channel ch1
2016-08-24 10:15:05,592 (conf-file-poller-0) [INFO - org.apache.flume.source.DefaultSourceFactory.create(DefaultSourceFactory.java:41)] Creating instance of source avro-source1, type avro
2016-08-24 10:15:05,610 (conf-file-poller-0) [INFO - org.apache.flume.sink.DefaultSinkFactory.create(DefaultSinkFactory.java:42)] Creating instance of sink: log-sink1, type: logger
2016-08-24 10:15:05,614 (conf-file-poller-0) [INFO - org.apache.flume.node.AbstractConfigurationProvider.getConfiguration(AbstractConfigurationProvider.java:114)] Channel ch1 connected to [avro-source1, log-sink1]
2016-08-24 10:15:05,623 (conf-file-poller-0) [INFO - org.apache.flume.node.Application.startAllComponents(Application.java:138)] Starting new configuration:{ sourceRunners:{avro-source1=EventDrivenSourceRunner: { source:Avro source avro-source1: { bindAddress: 10.10.1.23, port: 41414 } }} sinkRunners:{log-sink1=SinkRunner: { policy:org.apache.flume.sink.DefaultSinkProcessor@8ae0e27 counterGroup:{ name:null counters:{} } }} channels:{ch1=org.apache.flume.channel.MemoryChannel{name: ch1}} }
2016-08-24 10:15:05,632 (conf-file-poller-0) [INFO - org.apache.flume.node.Application.startAllComponents(Application.java:145)] Starting Channel ch1
2016-08-24 10:15:05,700 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.register(MonitoredCounterGroup.java:120)] Monitored counter group for type: CHANNEL, name: ch1: Successfully registered new MBean.
2016-08-24 10:15:05,701 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.start(MonitoredCounterGroup.java:96)] Component type: CHANNEL, name: ch1 started
2016-08-24 10:15:05,701 (conf-file-poller-0) [INFO - org.apache.flume.node.Application.startAllComponents(Application.java:173)] Starting Sink log-sink1
2016-08-24 10:15:05,702 (conf-file-poller-0) [INFO - org.apache.flume.node.Application.startAllComponents(Application.java:184)] Starting Source avro-source1
2016-08-24 10:15:05,702 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.source.AvroSource.start(AvroSource.java:228)] Starting Avro source avro-source1: { bindAddress: 10.10.1.23, port: 41414 }...
2016-08-24 10:15:05,702 (SinkRunner-PollingRunner-DefaultSinkProcessor) [DEBUG - org.apache.flume.SinkRunner$PollingRunner.run(SinkRunner.java:143)] Polling sink runner starting
2016-08-24 10:15:06,036 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.register(MonitoredCounterGroup.java:120)] Monitored counter group for type: SOURCE, name: avro-source1: Successfully registered new MBean.
2016-08-24 10:15:06,036 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.instrumentation.MonitoredCounterGroup.start(MonitoredCounterGroup.java:96)] Component type: SOURCE, name: avro-source1 started
2016-08-24 10:15:06,036 (lifecycleSupervisor-1-0) [INFO - org.apache.flume.source.AvroSource.start(AvroSource.java:253)] Avro source avro-source1 started.
```

#### 查看Flume监听的41414端口，等待AvroClient的输入信息
```
# 查看flume的PID
$ ps -ef | grep flume
  501  6931  6602   0 10:27AM ttys000    0:00.00 grep flume
  501  6734  4754   0 10:15AM ttys001    0:03.61 /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp /Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/* -Djava.library.path= org.apache.flume.node.Application -f conf/flume.conf -n agent1

# 查看一下在监听41414端口的PID
$ lsof -i:41414
COMMAND  PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    6734 yunyu  156u  IPv6 0xd0e0ada8c2f7ce75      0t0  TCP localhost:41414 (LISTEN)
```

#### 启动Flume AvroClient端，发送数据到Agent测试

```
# 这里我使用的是本机IP：10.10.1.23，如果是本机测试也可以使用localhost
# 这里的启动命令是./bin/flume-ng avro-client
# -H：AvroClient指定Flume-ng Agent的IP或者hostname
# -p：AvroClient指定Avro source正在监听的端口
# -f：发送指定文件的每行数据给Flume Agent
$ ./bin/flume-ng avro-client --conf conf -H 10.10.1.23 -p 41414 -F /etc/passwd -Dflume.root.logger=DEBUG,console

+ exec /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/bin/java -Xmx20m -Dflume.root.logger=DEBUG,console -cp '/Users/yunyu/dev/flume-1.6.0/conf:/Users/yunyu/dev/flume-1.6.0/lib/*' -Djava.library.path= org.apache.flume.client.avro.AvroCLIClient -H 10.10.1.23 -p 41414 -F /etc/passwd
2016-08-24 10:21:30,468 (main) [DEBUG - org.apache.flume.api.NettyAvroRpcClient.configure(NettyAvroRpcClient.java:499)] Batch size string = 5
2016-08-24 10:21:30,477 (main) [WARN - org.apache.flume.api.NettyAvroRpcClient.configure(NettyAvroRpcClient.java:634)] Using default maxIOWorkers
2016-08-24 10:21:30,887 (main) [DEBUG - org.apache.flume.client.avro.AvroCLIClient.run(AvroCLIClient.java:234)] Finished
2016-08-24 10:21:30,887 (main) [DEBUG - org.apache.flume.client.avro.AvroCLIClient.run(AvroCLIClient.java:237)] Closing reader
2016-08-24 10:21:30,890 (main) [DEBUG - org.apache.flume.client.avro.AvroCLIClient.run(AvroCLIClient.java:241)] Closing RPC client
2016-08-24 10:21:30,896 (main) [DEBUG - org.apache.flume.client.avro.AvroCLIClient.main(AvroCLIClient.java:84)] Exiting
```

#### Flume Agent端控制台

```
# Flume Agent端控制台会有类似如下的输出信息

2016-08-24 10:23:26,821 (New I/O server boss #1 ([id: 0x0b65e837, /10.10.1.23:41414])) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 => /10.10.1.23:41414] OPEN
2016-08-24 10:23:26,821 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 => /10.10.1.23:41414] BOUND: /10.10.1.23:41414
2016-08-24 10:23:26,821 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 => /10.10.1.23:41414] CONNECTED: /10.10.1.23:63992
2016-08-24 10:23:27,054 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,073 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,075 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,076 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,077 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,078 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,079 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,081 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,082 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,083 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,084 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,085 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,086 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,087 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,088 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,090 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,092 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,094 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,095 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 5 events.
2016-08-24 10:23:27,096 (New I/O  worker #2) [DEBUG - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:371)] Avro source avro-source1: Received avro event batch of 1 events.
2016-08-24 10:23:27,100 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 :> /10.10.1.23:41414] DISCONNECTED
2016-08-24 10:23:27,100 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 :> /10.10.1.23:41414] UNBOUND
2016-08-24 10:23:27,100 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.handleUpstream(NettyServer.java:171)] [id: 0x78d53a57, /10.10.1.23:63992 :> /10.10.1.23:41414] CLOSED
2016-08-24 10:23:27,100 (New I/O  worker #2) [INFO - org.apache.avro.ipc.NettyServer$NettyServerAvroHandler.channelClosed(NettyServer.java:209)] Connection to /10.10.1.23:63992 disconnected.
2016-08-24 10:23:28,981 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 23 23                                           ## }
2016-08-24 10:23:28,981 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 23 20 55 73 65 72 20 44 61 74 61 62 61 73 65    # User Database }
2016-08-24 10:23:28,982 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 23 20                                           #  }
```

#### 官网给出的flume-ng启动参数

##### flume-ng global options

Option						|Description		
----------				|--------------
--classpath,-C <cp>		|Append to the classpath
--conf,-c <conf>			|Use configs in <conf> directory
--dryrun,-d 				|Do not actually start Flume, just print the command
-Dproperty=value 		|Sets a JDK system property value

##### flume-ng agent options

Option						|Description		
----------				|--------------
--conf-file,-f <file>	|Indicates which configuration file you want to run with (required)
--name,-n <agentname>	|Indicates the name of agent on which we're running (required)

##### flume-ng avro-client options

Option							|Description		
----------					|--------------
--host,-H <hostname>		|Specifies the hostname of the Flume agent (may be localhost)
--port,-p <port>				|Specifies the port on which the Avro source is listening
--filename,-F <filename>	|Sends each line of <filename> to Flume (optional)
--headerFile,-F <file>		|Header file containing headers as key/value pairs on each new line

OK，大功告成 ^_^

参考文章：

- https://cwiki.apache.org/confluence/display/FLUME/Getting+Started
- http://shiyanjun.cn/archives/915.html