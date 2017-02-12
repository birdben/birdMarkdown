---
title: "Kafka学习（四）指定Kafka的data和logs路径"
date: 2016-12-15 11:16:39
tags: [Kafka]
categories: [MQ]
---

### 环境说明

- zookeeper-3.4.8
- kafka_2.11-0.9.0.0

最近公司在做日志收集系统，主要基于Elastic Stack做的，其中用到了Kafka服务器作缓冲，今天主要说的问题就是因为上了日志收集系统之后，大量的线上日志写入到Kafka，导致Kafka磁盘写满。后来发现是因为当时运维安装Kafka环境的时候将Kafka的data和logs都挂在了系统盘下，而且系统盘容量很小只有20G，所以出现了磁盘不够用的情况。

解决办法也很简单，将Kafka的data和logs分别写到挂在的数据磁盘即可。这里需要简单修改一下Kafka的配置文件。

#### 修改Kafka的data目录

Kafka的data目录是存储Kafka的数据文件的目录，是在${KAFKA_HOME}/config/server.properties中修改

```
log.dirs=/data/kafka_data
```

注意：log.dirs可以配置多个目录，需要用逗号分隔开

#### 修改Kafka的logs目录

Kafka运行的时候都会通过log4j打印很多日志文件，如：server.log, controller.log, state-change.log等，默认都会将其输出到${KAFKA_HOME}/logs目录下，这样很不利于线上运维，因为经常容易出现写满文件系统（尤其是挂在到系统盘的情况），所以一般运维都会建议将Kafka（或者其他环境）都安装在/usr/local系统盘下（方便clone系统镜像），一般系统盘都比较小，而数据和日志会指定到另一个或多个更大空间的挂在数据盘。

修改Kafka的logs目录是在${KAFKA_HOME}/bin/kafka-run-class.sh中修改

```
# Log directory to use
if [ "x$LOG_DIR" = "x" ]; then
    # LOG_DIR="$base_dir/logs"
    LOG_DIR="/data/kafka_logs"
fi
```
