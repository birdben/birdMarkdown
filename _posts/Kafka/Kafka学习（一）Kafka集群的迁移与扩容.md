---
title: "Kafka学习（一）Kafka集群的迁移与扩容"
date: 2016-12-26 16:40:47
tags: [Kafka]
categories: [MQ]
---

Kafka的集群扩容实际上就是把Topic的Partition移动到新加的集群节点上。我们只需要copy一份Kafka的安装目录到新的节点机器上，修改一下相关配置文件（broker.id，logs.dir等等），但是新添加的Kafka节点机器是不会自动分配数据的，所以需要我们手动操作。

Kafka的扩容可以细分为以下三种：

- 增加节点
- 增加Topic副本
- 增加分区

具体的步骤有两种方式:

- 通过--topics-to-move-json-file和--broker-list批量生成新的Topic分区信息，然后根据该信息执行转移操作。
- 手动写要移动的topic信息，更灵活，但是在大量Topic和Partition的情况下非常繁琐并且容易出错。

```
kafka-reassign-partitions.sh命令的三种模式

generate模式：给需要重新分配的Topic，自动生成reassign plan（执行计划），但并不执行
execute模式：根据指定的reassign plan（json文件）重新分配partition或者replication
verify模式：检查重新分配是否完成

$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --topics-to-move-json-file json_file --broker-list "brokerIds" --generate
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file json_file --execute
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file json_file --verify
```

### Kafka动态增加节点（Node）

新添加的Kafka节点机器是不会自动分配数据的，所以无法分担集群的负载，除非我们新建一个Topic，此时新的Topic会使用新添加的Kafka节点机器。如果我们想新添加的Kafka节点机器能够分担集群存储，需要手动将部分分区移动到新添加的Kafka节点机器上。

我们原来的Kafka节点分别是0，1，2，现在要加入3，4两个新节点

topicMove.json

```
{    "topics":[        {"topic":"logstash_test"}    ],    "version":1}
```

生成迁移的计划

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --topics-to-move-json-file topicMove.json --broker-list "0,1,2,3,4" --generateCurrent partition replica assignment{"version":1,"partitions":[{"topic":"logstash_test","partition":3,"replicas":[1,2,0]},{"topic":"logstash_test","partition":1,"replicas":[2,1,0]},{"topic":"logstash_test","partition":0,"replicas":[1,2,0]},{"topic":"logstash_test","partition":2,"replicas":[0,2,1]},{"topic":"logstash_test","partition":5,"replicas":[0,1,2]},{"topic":"logstash_test","partition":4,"replicas":[2,0,1]}]}Proposed partition reassignment configuration{"version":1,"partitions":[{"topic":"logstash_test","partition":3,"replicas":[2,0,1]},{"topic":"logstash_test","partition":1,"replicas":[0,3,4]},{"topic":"logstash_test","partition":0,"replicas":[4,2,3]},{"topic":"logstash_test","partition":2,"replicas":[1,4,0]},{"topic":"logstash_test","partition":5,"replicas":[4,3,0]},{"topic":"logstash_test","partition":4,"replicas":[3,1,2]}]}
```

上面是原来的所有partition在各个节点的分布情况，下面是加入4，5两个新节点之后所有partition在各个节点的分布情况

将生成的执行计划保存为add_node.json文件

重新分配partition（其实这里是添加新的节点）

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_node.json --executeCurrent partition replica assignment{"version":1,"partitions":[{"topic":"logstash_test","partition":3,"replicas":[1,2,0]},{"topic":"logstash_test","partition":1,"replicas":[2,1,0]},{"topic":"logstash_test","partition":0,"replicas":[1,2,0]},{"topic":"logstash_test","partition":2,"replicas":[0,2,1]},{"topic":"logstash_test","partition":5,"replicas":[0,1,2]},{"topic":"logstash_test","partition":4,"replicas":[2,0,1]}]}Save this to use as the --reassignment-json-file option during rollbackSuccessfully started reassignment of partitions {"version":1,"partitions":[{"topic":"logstash_test","partition":0,"replicas":[4,2,3]},{"topic":"logstash_test","partition":4,"replicas":[3,1,2]},{"topic":"logstash_test","partition":5,"replicas":[4,3,0]},{"topic":"logstash_test","partition":2,"replicas":[1,4,0]},{"topic":"logstash_test","partition":3,"replicas":[2,0,1]},{"topic":"logstash_test","partition":1,"replicas":[0,3,4]}]}
```

查看执行的状态

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_node.json --verify
```

```
ERROR: Assigned replicas (X,X) don't match the list of replicas for reassignment (X,X) for partition [logstash_test,1]
```

假设出现类似这样的错误，他并不是真的出错，而是指目前仍在复制数据中。再过一段时间再运行verify命令，他就会消失(加入完成拷贝)

当然也可以不使用generate先生成执行计划，而是自己直接手动编辑生成add_node.json文件内容，然后直接execute执行，但是这样容易出错，Kafka节点比较少的时候推荐使用。

注意：其实通过generate的结果我们也可以看出，其实不管是扩容，减容，迁移，其实都是重新分配Topic或者Partition的过程。json文件的内容都是指定Topic下的Partition要移动到哪个Node上。


### Kafka动态增加Topic副本（Replication）

查看当前node_log的Topic信息

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:1	ReplicationFactor:1	Configs:	Topic: node_log	Partition: 0	Leader: 2	Replicas: 2	Isr: 2
```

add_replication.json

```
{
    "version":1
    "partitions":[
        {
            "topic":"node_log",
            "partition":0,
            "replicas":[
                0,
                1,
                2
            ]
        }
    ]
}
```

重新分配partition（其实这里是添加副本replication）

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_replication.json --executeCurrent partition replica assignment{"version":1,"partitions":[{"topic":"node_log","partition":0,"replicas":[2]}]}Save this to use as the --reassignment-json-file option during rollbackSuccessfully started reassignment of partitions {"version":1,"partitions":[{"topic":"node_log","partition":0,"replicas":[0,1,2]}]}
```

查看执行的状态

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_replication.json --verify
Status of partition reassignment:Reassignment of partition [node_log,0] completed successfully
```

如果遇到下面的错误很有可能是json文件格式有错误，仔细检查修正重新运行即可

```
Partitions reassignment failed due to Partition reassignment data file add_replication.json is emptykafka.common.AdminCommandFailedException: Partition reassignment data file add_replication.json is empty	at kafka.admin.ReassignPartitionsCommand$.executeAssignment(ReassignPartitionsCommand.scala:120)	at kafka.admin.ReassignPartitionsCommand$.main(ReassignPartitionsCommand.scala:52)	at kafka.admin.ReassignPartitionsCommand.main(ReassignPartitionsCommand.scala)
```

查看添加replication后的node_log的Topic信息

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:1	ReplicationFactor:2	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 1,0,2
```

### Kafka动态增加分区（Parition）

查看当前node_log的Topic信息

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:1	ReplicationFactor:2	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 1,0,2
```

add_partition.json

```
{    "version":1,    "partitions":[        {            "topic":"node_log",            "partition":0,            "replicas":[                0,                1,                2            ]        },        {            "topic":"node_log",            "partition":1,            "replicas":[                0,                1,                2            ]        },        {            "topic":"node_log",            "partition":2,            "replicas":[                0,                1,                2            ]        }    ]}
```

重新分配partition

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_partition.json --executeCurrent partition replica assignment{"version":1,"partitions":[{"topic":"node_log","partition":0,"replicas":[0,1,2]}]}Save this to use as the --reassignment-json-file option during rollback[2016-12-30 12:22:17,195] ERROR Skipping reassignment of partition [node_log,1] since it doesn't exist (kafka.admin.ReassignPartitionsCommand)[2016-12-30 12:22:17,207] ERROR Skipping reassignment of partition [node_log,2] since it doesn't exist (kafka.admin.ReassignPartitionsCommand)Successfully started reassignment of partitions {"version":1,"partitions":[{"topic":"node_log","partition":0,"replicas":[0,1,2]},{"topic":"node_log","partition":1,"replicas":[0,1,2]},{"topic":"node_log","partition":2,"replicas":[0,1,2]}]}
```

查看执行的状态

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_partition.json --verify
Status of partition reassignment:ERROR: Assigned replicas (1,2,0) don't match the list of replicas for reassignment (0,1,2) for partition [node_log,1]ERROR: Assigned replicas (2,0,1) don't match the list of replicas for reassignment (0,1,2) for partition [node_log,2]Reassignment of partition [node_log,0] completed successfullyReassignment of partition [node_log,1] failedReassignment of partition [node_log,2] failed
```

报错是因为我们只有一个partition0，没有partition1，partition2，所以跳过了重新分配partition的过程。

我们需要先增加partition的数量，我们把partition的数量变成6个

```
$ kafka-topics.sh --zookeeper localhost:2181 --alter --topic node_log --partitions 6WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affectedAdding partitions succeeded!
```

查看添加partition后的node_log的Topic信息

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:6	ReplicationFactor:3	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 1,0,2	Topic: node_log	Partition: 1	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0	Topic: node_log	Partition: 2	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1	Topic: node_log	Partition: 3	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1	Topic: node_log	Partition: 4	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2	Topic: node_log	Partition: 5	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
```

重新执行reassign，然后查看执行状态

```
$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file ~/Downloads/add_partition.json --verifyStatus of partition reassignment:Reassignment of partition [node_log,0] completed successfullyReassignment of partition [node_log,1] completed successfullyReassignment of partition [node_log,2] completed successfully
```

重新分配执行完成之后，再次查看node_log的Topic信息

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:6	ReplicationFactor:3	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 1,2,0	Topic: node_log	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 1,2,0	Topic: node_log	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 1,2,0	Topic: node_log	Partition: 3	Leader: 0	Replicas: 0,2,1	Isr: 1,2,0	Topic: node_log	Partition: 4	Leader: 1	Replicas: 1,0,2	Isr: 1,2,0	Topic: node_log	Partition: 5	Leader: 2	Replicas: 2,1,0	Isr: 1,2,0
```

执行reassign之后，我们发现0-3的partition的leader都是0，这是因为我们之前突然扩展partition到6个，而我们在reassign的之后只指定了0-2的partition的分配，我们修改一下add_partition.json文件如下，将6个partition均匀的分布在我们3个broker节点上，每个partition有2个replication。

add_partition.json

```
{    "version":1,    "partitions":[        {            "topic":"node_log",            "partition":0,            "replicas":[                0,                1            ]        },        {            "topic":"node_log",            "partition":1,            "replicas":[                1,                2            ]        },        {            "topic":"node_log",            "partition":2,            "replicas":[                2,                0            ]        },        {            "topic":"node_log",            "partition":3,            "replicas":[                0,                1            ]        },        {            "topic":"node_log",            "partition":4,            "replicas":[                1,                2            ]        },        {            "topic":"node_log",            "partition":5,            "replicas":[                2,                0            ]        }    ]}
```

再次查看reassign之后的Topic情况

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:6	ReplicationFactor:2	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1	Topic: node_log	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 1,2	Topic: node_log	Partition: 2	Leader: 0	Replicas: 2,0	Isr: 0,2	Topic: node_log	Partition: 3	Leader: 0	Replicas: 0,1	Isr: 0,1	Topic: node_log	Partition: 4	Leader: 1	Replicas: 1,2	Isr: 1,2	Topic: node_log	Partition: 5	Leader: 2	Replicas: 2,0	Isr: 0,2
```


我们发现还是有点小问题，partition2的leader仍然是0。这里先普及一下assigned replicas和preferred replica

#### assigned replicas和preferred replica

每个partitiion的所有replicas叫做"assigned replicas"，"assigned replicas"中的第一个replicas叫"preferred replica"，刚创建的topic一般"preferred replica"是leader。leader replica负责所有的读写。但随着时间推移，broker可能会停机，会导致leader迁移，导致机群的负载不均衡。

我们这里preferred replica已经是2了，但是leader却不是2，这样我们需要重新选举一下leader，需要使用kafka-preferred-replica-election.sh来调整。

两种操作方式：

- 对所有Topics进行操作

```
$ kafka-preferred-replica-election.sh --zookeeper localhost:2181
```

- 对某个Topic进行操作（json文件中指定要操作的Topic）

```
$ kafka-preferred-replica-election.sh --zookeeper localhost:2181 --path-to-json-file json_file
```

注意：这里我们只需要调整node_log的Topic的partition2的leader。

leaderTopic.json

```
{ "partitions":  [    {"topic":"node_log","partition":2}  ]}
```

重新选举leader

```
$ kafka-preferred-replica-election.sh --zookeeper localhost:2181 --path-to-json-file leaderTopic.jsonSuccessfully started preferred replica election for partitions Set([node_log,2])
```

再次查看Topic情况，发现leader也是我们所期望的均匀分配了

```
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic node_logTopic:node_log	PartitionCount:6	ReplicationFactor:2	Configs:	Topic: node_log	Partition: 0	Leader: 0	Replicas: 0,1	Isr: 0,1	Topic: node_log	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 2,1	Topic: node_log	Partition: 2	Leader: 2	Replicas: 2,0	Isr: 2,0	Topic: node_log	Partition: 3	Leader: 0	Replicas: 0,1	Isr: 0,1	Topic: node_log	Partition: 4	Leader: 1	Replicas: 1,2	Isr: 2,1	Topic: node_log	Partition: 5	Leader: 2	Replicas: 2,0	Isr: 0,2
```

数据迁移的步骤

1. The tool updates the zookeeper path “/admin/reassign_partitions” with the list of topic partitions and (if specified in the Json file) the list of their new assigned replicas.

2. The controller listens to the path above. When a data change update is triggered, the controller reads the list of topic partitions and their assigned replicas from zookeeper.

3. For each topic partition, the controller does the following:

* 3.1. Start new replicas in RAR – AR (RAR = Reassigned Replicas, AR = original list of Assigned Replicas)
* 3.2. Wait until new replicas are in sync with the leader
* 3.3. If the leader is not in RAR, elect a new leader from RAR
* 3.4. Stop old replicas AR – RAR
* 3.5. Write new AR
* 3.6. Remove partition from the /admin/reassign_partitions path

参考文章：

- http://blog.csdn.net/lsshlsw/article/details/47615747
- https://www.iteblog.com/archives/1611
- https://www.iteblog.com/archives/1613
- https://www.iteblog.com/archives/1614
- http://wzktravel.github.io/2015/12/31/kafka-reassign/
- http://blog.csdn.net/lizhitao/article/details/41441513
- https://cwiki.apache.org/confluence/display/KAFKA/Replication+tools#Replicationtools-2.PreferredReplicaLeaderElectionTool
- https://cwiki.apache.org/confluence/pages/diffpages.action?pageId=31819985&originalId=34022604