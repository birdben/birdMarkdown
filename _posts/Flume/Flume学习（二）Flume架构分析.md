---
title: "Flume学习（二）Flume架构分析"
date: 2016-08-24 14:14:59
tags: [Flume]
categories: [Log]
---

#### Flume NG架构

##### 基础架构
![基础架构](http://flume.apache.org/_images/UserGuide_image00.png)

##### 多个Agent顺序连接
可以将多个Agent顺序连接起来，将最初的数据源经过收集，存储到最终的存储系统中。这是最简单的情况，一般情况下，应该控制这种顺序连接的Agent的数量，因为数据流经的路径变长了，如果不考虑failover的话，出现故障将影响整个Flow上的Agent收集服务。

![多个Agent顺序连接](http://7xnrdo.com1.z0.glb.clouddn.com/2014/flume-multiseq-agents.png)

##### 多个Agent的数据汇聚到同一个Agent
这种情况应用的场景比较多，比如要收集Web网站的用户行为日志，Web网站为了可用性使用的负载均衡的集群模式，每个节点都产生用户行为日志，可以为每个节点都配置一个Agent来单独收集日志数据，然后多个Agent将数据最终汇聚到一个用来存储数据存储系统，如HDFS上。

![多个Agent的数据汇聚到同一个Agent](http://7xnrdo.com1.z0.glb.clouddn.com/2014/flume-join-agent.png)

##### 多路（Multiplexing）Agent
这种模式，有两种方式，一种是用来复制（Replication），另一种是用来分流（Multiplexing）。Replication方式，可以将最前端的数据源复制多份，分别传递到多个channel中，每个channel接收到的数据都是相同的，配置格式，如下所示：

```
# List the sources, sinks and channels for the agent
<Agent>.sources = <Source1>
<Agent>.sinks = <Sink1> <Sink2>
<Agent>.channels = <Channel1> <Channel2>

# set list of channels for source (separated by space)
<Agent>.sources.<Source1>.channels = <Channel1> <Channel2>

# set channel for sinks
<Agent>.sinks.<Sink1>.channel = <Channel1>
<Agent>.sinks.<Sink2>.channel = <Channel2>
<Agent>.sources.<Source1>.selector.type = replicating
```

上面指定了selector的type的值为replication，其他的配置没有指定，使用的Replication方式，Source1会将数据分别存储到Channel1和Channel2，这两个channel里面存储的数据是相同的，然后数据被传递到Sink1和Sink2。
Multiplexing方式，selector可以根据header的值来确定数据传递到哪一个channel，配置格式，如下所示：

```
# Mapping for multiplexing selector
<Agent>.sources.<Source1>.selector.type = multiplexing
<Agent>.sources.<Source1>.selector.header = <someHeader>
<Agent>.sources.<Source1>.selector.mapping.<Value1> = <Channel1>
<Agent>.sources.<Source1>.selector.mapping.<Value2> = <Channel1> <Channel2>
<Agent>.sources.<Source1>.selector.mapping.<Value3> = <Channel2>

<Agent>.sources.<Source1>.selector.default = <Channel2>
```

上面selector的type的值为multiplexing，同时配置selector的header信息，还配置了多个selector的mapping的值，即header的值：如果header的值为Value1、Value2，数据从Source1路由到Channel1；如果header的值为Value2、Value3，数据从Source1路由到Channel2。

![多路（Multiplexing）Agent](http://7xnrdo.com1.z0.glb.clouddn.com/2014/flume-multiplexing-agent.png)

##### 实现load balance功能
Load balancing Sink Processor能够实现load balance功能，下图Agent1是一个路由节点，负责将Channel暂存的Event均衡到对应的多个Sink组件上，而每个Sink组件分别连接到一个独立的Agent上，示例配置，如下所示：

```
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2 k3
a1.sinkgroups.g1.processor.type = load_balance
a1.sinkgroups.g1.processor.backoff = true
a1.sinkgroups.g1.processor.selector = round_robin
a1.sinkgroups.g1.processor.selector.maxTimeOut=10000
```

![实现load balance功能](http://7xnrdo.com1.z0.glb.clouddn.com/2014/flume-load-balance-agents.png)

##### 实现failover功能
Failover Sink Processor能够实现failover功能，具体流程类似load balance，但是内部处理机制与load balance完全不同：Failover Sink Processor维护一个优先级Sink组件列表，只要有一个Sink组件可用，Event就被传递到下一个组件。如果一个Sink能够成功处理Event，则会加入到一个Pool中，否则会被移出Pool并计算失败次数，设置一个惩罚因子，示例配置如下所示：

```
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2 k3
a1.sinkgroups.g1.processor.type = failover
a1.sinkgroups.g1.processor.priority.k1 = 5
a1.sinkgroups.g1.processor.priority.k2 = 7
a1.sinkgroups.g1.processor.priority.k3 = 6
a1.sinkgroups.g1.processor.maxpenalty = 20000
```


参考文章：

- http://shiyanjun.cn/archives/915.html
- http://blog.javachen.com/2014/07/22/flume-ng.html
- http://flume.apache.org/FlumeUserGuide.html