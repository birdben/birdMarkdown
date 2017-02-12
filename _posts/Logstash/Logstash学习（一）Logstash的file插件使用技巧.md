---
title: "Logstash学习（一）Logstash的file插件使用技巧"
date: 2016-12-21 16:19:46
tags: [Logstash]
categories: [Log]
---

最近在使用Logstash做Shipper端收集go，node，php多种日志。

```
input {
    file {
        path => ["/home/yunyu/Downloads/*.log.*", "/home/yunyu/Downloads/lumen.log"]
        codec => "plain"
        # 多个日志文件的offset信息都会记录到这个sincedb文件中，会记录成多行
        sincedb_path => "/data/logstash_sincedb/php/.sincedb_php"
        start_position => "beginning"
        # 设置是否忽略太旧的日志的
        # 如果没设置该属性可能会导致读取不到文件内容，因为我们的日志大部分是好几个月前的，所以这里设置为不忽略
        ignore_older => 0
    }
}

output {
    stdout {
        codec => rubydebug
    }
    kafka {
        # 指定Kafka集群地址
        bootstrap_servers => "hadoop1:9092,hadoop2:9092,hadoop3:9092"
        # 指定Kafka的Topic
        topic_id => "php_log"
    }
}

# path : 指定要采集的日志文件路径，多个日志文件可以使用通配符，也可以使用数组
# sincedb_path : sincedb文件是用于存储Logstash读取文件的位置，每行表示一个文件，每行有两个数字，第一个表示文件的inode，第二个表示文件读取到的位置（byteoffset），默认为$HOME/.sincedb*，文件名是日志文件路径MD5加密后的结果。sincedb_path只能指定为具体的file文件，不能是path目录。
# sincedb_write_interval : Logstash每隔多久写一次sincedb文件，默认是15秒。
# ignore_older : 在每次检查文件列表的时候，如果一个文件的最后修改时间超过这个值，就忽略这个文件。默认是86400秒，即一天。
# start_position : Logstash从什么位置开始读取文件数据，默认是结束位置，也就是说Logstash进程会以类似tail -F的形式运行。如果你是要导入原有数据，把这个设定改成"beginning"，logstash进程就从头开始读取。
```

这里php日志文件按照类型拆分成多个日志文件，这些日志文件都需要收集，所以需要我们在Logstash配置path使用了通配符来处理读取多个日志文件。这里和之前读取go和node日志不同，go和node的日志通常只有一个日志文件，这里php的日志文件按照类型分别写入到不同的日志文件，那我们指定的sincedb_path又只能指定一个file而不是path，那如何记录多个文件的读取进度呢？我在日志系统收集的过程中特意查看了一下sincedb文件，发现是如果Logstash file path指定了读取多个文件，这样sincedb文件就会存储多行，每行代表一个日志文件的读取进度。

```
# cat /data/logstash_sincedb/php/.sincedb_php
1058198 0 51713 976040
1058229 0 51713 151513
1056857 0 51713 970977345
1058215 0 51713 115587
1057737 0 51713 873729227
1058197 0 51713 1108533
1058230 0 51713 595155
1058110 0 51713 1085851
1048702 0 51713 10036
1057753 0 51713 206711384
1057777 0 51713 1607939699
1052414 0 51713 359573
1056858 0 51713 1015625306
1058201 0 51713 341179
1058233 0 51713 449755
1048838 0 51713 90406
1058196 0 51713 234060588

# 这四列的内容分别是：inode, major number, minor number, pos
# 可以参考文章：
# http://unix.stackexchange.com/questions/73988/linux-major-and-minor-device-numbers
```


参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-inputs-file.html#plugins-inputs-file-sincedb_path
- http://kibana.logstash.es/content/logstash/plugins/input/file.html
- http://blog.csdn.net/hengyunabc/article/details/25665877
- http://www.voidcn.com/blog/sdulibh/article/p-6234930.html