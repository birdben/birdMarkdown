---
title: "Kafka学习（七）Kafka命令使用"
date: 2017-07-04 12:27:59
tags: [Kafka]
categories: [MQ]
---

### kafka-topics.sh Kafka主题信息

```
# 创建新的Topic kafka_cluster_topic
# replication-factor : 备份为3
# partitions : 分区为1
$ kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic kafka_cluster_topic

# 查看Topic kafka_cluster_topic的状态，发现Leader是1（broker.id=1）,有三个备份分别是0，1，2
$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic kafka_cluster_topic
Topic:kafka_cluster_topic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: kafka_cluster_topic	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2	
	
# leader：是随机挑选出来的
# replicas：是负责同步leader的log的备份节点列表
# isr：是备份节点列表的子集，表示正在进行同步log的工作状态的节点列表

# 通过kafka-topics.sh工具的alter命令，将kafka_cluster_topic的partitions从1增加到5； 
$ kafka-topics.sh –-zookeeper localhost:2181 --alter –-partitions 5 -–topic kafka_cluster_topic
```

### kafka-console-producer.sh 控制台生产Kafka数据

```
# 启动producer服务，向test1的Topic中发送消息
$ kafka-console-producer.sh --broker-list localhost:9092 --topic test1
this is a message
this is another message
still a message
```

### kafka-console-consumer.sh 控制台消费Kafka数据

```
# 启动consumer服务，从test1的Topic中接收消息
$ kafka-console-consumer.sh --zookeeper localhost:2181 --topic test1 --from-beginning
this is a message
this is another message
still a message
```

### kafka-consumer-groups.sh Kafka消费者组信息

```
# 适用于Offset存储于Kafka（使用--bootstrap-server参数必须带有--new-consumer参数）
# 查看所有group
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list --new-consumer
data_flow_group_dev_host_v1
data_flow_group_product_host_v1

# 提示delete参数只对zookeeper参数有效，也就是说Offset存储于Kafka的情况，目前是无法手动删除group的
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --delete --group data_flow_group_dev_host_v1 --topic host_log
Option '[delete]' is only valid with '[zookeeper]'. Note that there's no need to delete group metadata for the new consumer as the group is deleted when the last committed offset for that group expires.

# 适用于Offset存储于Zookeeper
# 查看所有group
$ kafka-consumer-groups.sh --zookeeper localhost:2181 --list

# 删除指定的group（WARNING: Group deletion only works for old ZK-based consumer groups, and one has to use it carefully to only delete groups that are not active.）
$ kafka-consumer-groups.sh --zookeeper localhost:2181 --delete --group logstash_go
```

### kafka-simple-consumer-shell.sh Kafka简单消费者

- 使用Simple Consumer API从指定Topic的分区读取数据并打印在终端

```
$ kafka-simple-consumer-shell.sh --broker-list localhost:9092 --topic host_log --partition 0
```

- 查看保存在Kafka内部的offset

```
# 查看所有group
$ kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list --new-consumer
```

找到指定的group之后，根据group的名称计算hash值。这里一定是%50因为默认情况下__consumer_offsets有50个分区，在kafka_logs目录查看的__consumer_offsets就是0-49，一共50个

```
public static void main(String[] args) {
    int i = Math.abs("data_flow_group_dev_host_v1".hashCode()) % 50;
    System.out.println(i);
}

结果是44
```

然后使用kafka-simple-consumer-shell.sh消费host_log主题的partition 44，可以看到Offset每隔一段时间就会记录一条日志。

```
$ kafka-simple-consumer-shell.sh --broker-list localhost:9092 --topic __consumer_offsets --partition 44 --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter"
...
[data_flow_group_dev_host_v1,host_log,0]::[OffsetMetadata[2865485,NO_METADATA],CommitTime 1499159432215,ExpirationTime 1499245832215]
[data_flow_group_dev_host_v1,host_log,0]::[OffsetMetadata[2865485,NO_METADATA],CommitTime 1499159463262,ExpirationTime 1499245863262]
[data_flow_group_dev_host_v1,host_log,0]::[OffsetMetadata[2865485,NO_METADATA],CommitTime 1499159494307,ExpirationTime 1499245894307]
[data_flow_group_dev_host_v1,host_log,0]::[OffsetMetadata[2865485,NO_METADATA],CommitTime 1499159525353,ExpirationTime 1499245925353]
```

### kafka-run-class.sh Kafka相关工具

##### kafka.tools.ConsumerOffsetChecker：查看指定消费者组的offset信息

```
# 适用于Offset存储于Zookeeper
# 列出了所有消费者组的所有信息：Group（消费者组），Topic（主题），Pid（分区ID），Offset（当前已消费的条数），LogSize（总条数），Lag（未消费的条数），Owner（指定的主题和消费者组的所有者）
$ kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --topic log_go --group logstash_go
[2017-07-04 16:19:49,132] WARN WARNING: ConsumerOffsetChecker is deprecated and will be dropped in releases following 0.9.0. Use ConsumerGroupCommand instead. (kafka.tools.ConsumerOffsetChecker$)
Group           Topic                          Pid Offset          logSize         Lag             Owner
logstash_go     log_go                         0   346117935       346117936       1               logstash_go_yy-log-es01-1495705743028-2ae56197-0
logstash_go     log_go                         1   277333911       277333911       0               logstash_go_yy-log-es01-1495705743028-2ae56197-0
logstash_go     log_go                         2   277317376       277317377       1               logstash_go_yy-log-es01-1495705743028-2ae56197-0
logstash_go     log_go                         3   277332996       277332997       1               logstash_go_yy-log-es01-1495705743028-2ae56197-0
logstash_go     log_go                         4   277333914       277333914       0               logstash_go_yy-log-es01-1495705743028-2ae56197-0
logstash_go     log_go                         5   277317570       277317570       0               logstash_go_yy-log-es01-1495705743028-2ae56197-0
```

##### kafka.tools.GetOffsetShell：查询topic的offset的范围

```
# 适用于Offset存储于Zookeeper和Kafka

# 查询offset的最大值
$ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic host_log --time -1
host_log:0:2865485

# 查询offset的最小值
$ kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic host_log --time -2
host_log:0:0
```

从上面的输出可以看出topic:host_log只有一个partition:0，offset范围为:[0, 2865485]

##### kafka.tools.UpdateOffsetsInZK：手动更新Kafka存在Zookeeper中的偏移量

需要配置consumer.properties文件如下：

```
zookeeper.connect=localhost:2181

# timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=6000

# consumer group id
group.id=logstash_go
```

```
$ kafka-run-class.sh kafka.tools.UpdateOffsetsInZK earliest config/consumer.properties log_go
updating partition 0 with new offset: 1351138
```

> 注意：这个工具只适用Kafka 0.8.x版本，新版本的Kafka已经将Offset默认存储在Kafka本地，而不是存储在Zookeeper中。

这个工具只能把Zookeeper中偏移量设置成earliest或者latest。如果要手动设置Offset为指定的值，可以手动修改Zookeeper中存储的Offset值。

```
# Offset在Zookeeper中的存储路径
/consumers/[groupId]/offsets/[topic]/[partitionId]

# 连接ZooKeeper
$ zkCli.sh -server localhost:2181

# 查看当前log_go的分区0的Offset值
[zk: localhost:2181(CONNECTED) 1] get /consumers/logstash_go/offsets/log_go/0
346113720
cZxid = 0xc000b4e24
ctime = Thu Dec 22 11:57:00 CST 2016
mZxid = 0x110916e0ba
mtime = Tue Jul 04 16:04:12 CST 2017
pZxid = 0xc000b4e24
cversion = 0
dataVersion = 6686701
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0

# 设置log_go的分区0的Offset为0
[zk: localhost:2181(CONNECTED) 1] set /consumers/logstash_go/offsets/log_go/0 0
cZxid = 0xc000b4e24
ctime = Thu Dec 22 11:57:00 CST 2016
mZxid = 0x11094a1c2b
mtime = Tue Jul 04 16:05:45 CST 2017
pZxid = 0xc000b4e24
cversion = 0
dataVersion = 6686704
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
```

##### kafka.tools.DumpLogSegments：直接查看Kafka的log文件中的信息

```
$ kafka-run-class.sh kafka.tools.DumpLogSegments --files /usr/local/kafka/data/host_log-0/00000000000000000000.log  --print-data-log

# 可以结合grep使用
$ kafka-run-class.sh kafka.tools.DumpLogSegments --files /usr/local/kafka/data/host_log-0/00000000000000000000.log  --print-data-log  | grep -e "111729662"

$ kafka-run-class.sh kafka.tools.DumpLogSegments --files /data0/kafka_data/host_log-0/00000000000000000000.log  --print-data-log  | grep -e "2017-06-19T15:5"

# 也可以将查询的指定结果输出到新的文件中
$ kafka-run-class.sh kafka.tools.DumpLogSegments --files /usr/local/kafka/data/host_log-0/00000000000000000000.log --print-data-log > /root/host_log_20170619.log
```

### kafka-reassign-partitions.sh 重新分配分区

```
kafka-reassign-partitions.sh命令的三种模式

generate模式：给需要重新分配的Topic，自动生成reassign plan（执行计划），但并不执行
execute模式：根据指定的reassign plan（json文件）重新分配partition或者replication
verify模式：检查重新分配是否完成

$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --topics-to-move-json-file json_file --broker-list "brokerIds" --generate

$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file json_file --execute

$ kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file json_file --verify
```

- Kafka动态增加节点（Node）
- Kafka动态增加Topic副本（Replication）
- Kafka动态增加分区（Parition）

具体细节请查看《Kafka学习（六）Kafka集群的迁移与扩容》

### kafka-preferred-replica-election.sh 重新选举leader

- 对所有Topics进行操作

```
$ kafka-preferred-replica-election.sh --zookeeper localhost:2181
```

- 对某个Topic进行操作（json文件中指定要操作的Topic）

```
$ kafka-preferred-replica-election.sh --zookeeper localhost:2181 --path-to-json-file json_file
```

具体细节请查看《Kafka学习（六）Kafka集群的迁移与扩容》

参考文章：

- https://cwiki.apache.org/confluence/display/KAFKA/System+Tools
- https://cwiki.apache.org/confluence/display/KAFKA/Replication+tools
- http://blog.csdn.net/xiaoyu_bd/article/details/52398265
- https://www.iteblog.com/archives/1642.html
- https://stackoverflow.com/questions/29243896/removing-a-kafka-consumer-group-in-zookeeper
- https://mail-archives.apache.org/mod_mbox/kafka-users/201601.mbox/%3CCAAUywg9kwiJ5kxzbjeubQv6KDV2K07x9XZq4r+mFNGfwgpbVTA@mail.gmail.com%3E
- http://www.cnblogs.com/huxi2b/p/6223228.html
- http://www.cnblogs.com/huxi2b/p/6061110.html