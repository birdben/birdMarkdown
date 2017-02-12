---
title: "Kafka学习（一）KafkaManager工具使用"
date: 2016-12-15 11:16:39
tags: [Kafka]
categories: [MQ]
---

```
$ kfka-run-class.sh kafka.tools.ConsumerOffsetChecker --zkconnect=192.168.199.129:2181,192.168.199.130:2181,192.168.199.131:2181 --group=group-1 
执行结果如下：列出了所有消费者组的所有信息，包括Group(消费者组)、Topic、Pid(分区id)、Offset(当前已消费的条数)、LogSize(总条数)、Lag(未消费的条数)、Owner 
```

参考文章：

- https://www.iteblog.com/archives/1084
- https://www.iteblog.com/archives/1083
- http://www.cnblogs.com/wangb0402/p/6227327.html