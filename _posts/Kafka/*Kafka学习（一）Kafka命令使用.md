---
title: "Kafka学习（一）Kafka命令使用"
date: 2016-12-15 11:16:39
tags: [Kafka]
categories: [MQ]
---

```
$ kfka-run-class.sh kafka.tools.ConsumerOffsetChecker --zkconnect=192.168.199.129:2181,192.168.199.130:2181,192.168.199.131:2181 --group=group-1 
执行结果如下：列出了所有消费者组的所有信息，包括Group(消费者组)、Topic、Pid(分区id)、Offset(当前已消费的条数)、LogSize(总条数)、Lag(未消费的条数)、Owner 
```

```
./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic adlog --from-beginning


./bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group logstash --topic adlog
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (zh_CN.UTF-8)
[2016-12-02 15:08:34,444] WARN WARNING: ConsumerOffsetChecker is deprecated and will be dropped in releases following 0.9.0. Use ConsumerGroupCommand instead. (kafka.tools.ConsumerOffsetChecker$)
Group           Topic                          Pid Offset          logSize         Lag             Owner
logstash        adlog                          0   4669571         8219362         3549791         none
root@zk1:~/kafka_2.11-0.9.0.1#


./bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper localhost:2181 --group logstash --topic adlog
[2016-12-05 14:05:22,379] WARN WARNING: ConsumerOffsetChecker is deprecated and will be dropped in releases following 0.9.0. Use ConsumerGroupCommand instead. (kafka.tools.ConsumerOffsetChecker$)
Group           Topic                          Pid Offset          logSize         Lag             Owner
logstash        adlog                          0   9381263         10651036        1269773         none

./bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /root/kafka_data/adlog-0/00000000000009776947.log --print-data-log > /root/adlog_kafka_20161205.log

./bin/kafka-run-class.sh kafka.admin.ShutdownBroker --zookeeper localhost:2181 --broker #brokerId# --num.retries 3 --retry.interval.ms 60

./bin/kafka-run-class.sh kafka.admin.ShutdownBroker --zookeeper localhost:2181 --broker 2 --num.retries 3 --retry.interval.ms 60

Kafka0.8.2之后已经删除掉kafka.admin.ShutdownBroker

Isr（Leader会跟踪与其保持同步的Replica列表，该列表称为ISR（即in-sync Replica）。如果一个Follower宕机，或者落后太多，Leader将把它从ISR中移除。）
```

查看保存在Kafka内部的offset

```
$ bin/kafka-consumer-groups.sh --bootstrap-server 10.10.1.8:9092 --list --new-consumer
```

这里一定是%50因为默认情况下__consumer_offsets有50个分区，在kafka_logs目录查看的__consumer_offsets就是0-49，一共50个

```
public static void main(String[] args) {
    int i = Math.abs("aaaaa".hashCode()) % 50;
    System.out.println(i);
}

结果是35
```

```
$ ./bin/kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 35 --broker-list localhost:9092 --formatter "kafka.coordinator.GroupMetadataManager\$OffsetsMessageFormatter"
```

参考文章：

- http://flychao88.iteye.com/blog/2263163
- https://www.iteblog.com/archives/1605
- https://www.iteblog.com/archives/1642
- https://www.iteblog.com/archives/1384
- https://cwiki.apache.org/confluence/display/KAFKA/Replication+tools
- https://cwiki.apache.org/confluence/pages/diffpages.action?pageId=31819985&originalId=34022604
- http://blog.csdn.net/damacheng/article/details/42393865
- http://blog.csdn.net/damacheng/article/details/42393859
- https://discuss.elastic.co/t/multiple-logstash-reading-from-a-single-kafka-topic/27727/10
- http://blog.csdn.net/lizhitao/article/details/45894245
- http://blog.csdn.net/lizhitao/article/details/45894109
- http://blog.csdn.net/lizhitao/article/details/45894189
- http://www.cnblogs.com/huxi2b/p/6061110.html
- http://www.cnblogs.com/huxi2b/p/6223228.html