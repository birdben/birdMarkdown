---
title: "Logstash学习（八）Logstash的metrics告警使用"
date: 2017-02-15 17:53:16
tags: [Logstash]
categories: [Log]
---

最近为了提高系统的运行稳定性，在日志收集的过程中要求添加错误日志的告警，这里主要使用Logstash自带的metrics功能。Logstash可以在filter中根据某些字段进行日志的分类，如果某一类的日志出现次数不在正常范围，就会触发metrics event然后进行告警操作，这里我们只是使用简单的发邮件的告警方式。

Java的日志格式

```
2017-02-14 14:36:40 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:41 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:42 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:43 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:44 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:45 [ INFO] - com.yunyu.birdben.task.RiskTask -RiskTask.java(97) -我是日志信息
2017-02-14 14:36:46 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:47 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:48 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:49 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:50 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:51 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:52 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
2017-02-14 14:36:53 [ INFO] - com.yunyu.birdben.task.OtherTask -OtherTask.java(97) -我是日志信息
```

Grok表达式

```
JAVA_TIMESTAMP %{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME}
JAVA_LOGS %{JAVA_TIMESTAMP:timestamp} \[ %{DATA:level}\] - %{DATA:class_name} -%{DATA:file_name}.java\(%{DATA:line}\) -%{GREEDYDATA:msg}
```

日志中有两个文件RiskTask.java和OtherTask.java文件，我们的需求是5分钟内，如果RiskTask的日志一条都没有出现就发送告警邮件。

这里使用了三个新的插件metrics，ruby，email

- metrics : 用来定时统计和生成metrics event的
- ruby : 使用ruby代码来定制metrics event失效的条件
- email : 不需要多说，就是用来发送告警邮件的

#### metrics插件

默认情况下或根据flush_interval，每5秒刷新一次指标。 指标在事件流中显示为新事件，并执行发生在事件流以及输出之后的任何过滤器。

一般来说，您需要为指标添加标记，并让输出显式查找该标记。

被刷新的事件将以以下方式包括每个计量器和计时器度量：

- meter : 计量器度量

meter => [ "event_%{field_name}" ]

```
"[event_%{field_name}] [count]" - 事件的总数
"[event_%{field_name}] [rate_1m]" - 1分钟滑动窗口中的每秒事件率
"[event_%{field_name}] [rate_5m]" - 5分钟滑动窗口中的每秒事件率
"[event_%{field_name}] [rate_15m]" - 15分钟滑动窗口中的每秒事件率
```

- timer : 计时器度量

timer => [ "thing", "%{duration}" ]

```
"[thing] [count]" - 事件的总数
"[thing] [rate_1m]" - 1分钟滑动窗口中的每秒事件率
"[thing] [rate_5m]" - 5分钟滑动窗口中的每秒事件率
"[thing] [rate_15m]" - 15分钟滑动窗口中的每秒事件率
"[thing] [min]" - 此指标的最小值
"[thing] [max]" - 此指标的最大值
"[thing] [stddev]" - 此指标的标准差
"[thing] [mean]" - 这个指标的平均值
"[thing] [pXX]" - 此指标的第XX个百分位数（请参阅百分位数）
```

```
metrics {
    # 定义metrics计数器数据保存的字段名 field_name的值就是上面Grok表达式解析出来的字段名
    meter => [ "event_%{field_name}" ]
    # 给该metrics添加tag标签，用于区分metrics
    add_tag => [ "metric" ]
    # 每隔5分钟统计一次（测试环境可以适当改小）
    flush_interval => 300
    # 每隔5分钟（flush_interval + 1秒）清空计数器（测试环境可以适当改小）
    clear_interval => 301
    # 10秒内的message数据才统计，避免延迟
    ignore_older_than => 10
}
```

#### ruby插件

```
if "metric" in [tags] {
    ruby {
        # code是定义metrics的过滤规则，满足什么条件删除metric event日志
        # 如果code为空，就是metric event不会被cancel，那么最终metric event会output到elasticsearch/stdout/email，如果不想每个metric event都触发告警事件，就只能通过ruby插件的code添加ruby代码来控制metric event的取消条件
        # code => ""
        # 如果status_code是500的日志count小于100条，就忽略此事件(即不发送任何消息)。
        code => "event.cancel if event['event_500']['count'] < 100"
    }
}
```

#### email插件

```
# 测试环境建议注释掉邮件发送，否则邮箱容易爆炸
email {
    # stmp服务器地址
    address => "smtpdm.aliyun.com"
    # 发件人邮箱地址
    username => "service@post.XXX.com"
    # 发件人邮箱密码
    password => "123456"
    # 发件人邮箱
    from => "service@post.XXX.com"
    # 收件人邮箱
    to => "birdben@XXX.com"
    # 邮件主题
    subject => "告警：风控任务未执行"
    # 邮件内容
    htmlbody => "告警内容：com.yunyu.birdben.task.RiskTask没有执行"
}
```

总结一下我所理解的metrics原理：

配置文件定义好metrics之后，Logstash每隔flush_interval设置的时间就会自动创建一个metrics event，可以把metrics event理解成是Logstash自己创建的一条新的日志，这条新的日志有个名称是event_%{field_name}的字段（可能是event_A，event_B，field_name根据Grok表达式解析出来的结果确定的），event_%{field_name}的字段下有四个字段

- "[event_%{field_name}] [count]" - 事件的总数
- "[event_%{field_name}] [rate_1m]" - 1分钟滑动窗口中的每秒事件率
- "[event_%{field_name}] [rate_5m]" - 5分钟滑动窗口中的每秒事件率
- "[event_%{field_name}] [rate_15m]" - 15分钟滑动窗口中的每秒事件率

我们可以根据count（事件的总数）的值，来统计每隔flush_interval时间，我们要统计的event_%{field_name}日志的数量。举个例子，如果field_name是status_code，那我们要统计的日志就是event_200，event_302，event_400，event_500等等。那么，event_200.count就是每隔flush_interval时间内，stats_code是200的事件个数，其他同理。如果metrics event被保存到ES索引中，那么查看到的ES结果就会类似下面的结构。

```
"_source": {

    "@version": "1",
    "@timestamp": "2017-02-15T11:06:37.402Z",
    "message": "hadoop1",
    "evnet_500": {
		 "count": 17,
	    "rate_1m": 3.4,
	    "rate_5m": 3.4,
	    "rate_15m": 3.4
    },
    "evnet_302": {
	    "count": 1074,
	    "rate_1m": 197.62554026237865,
	    "rate_5m": 211.24966828088344,
	    "rate_15m": 213.60997535145182
    },
    "event_200": {
	    "count": 10483,
	    "rate_1m": 982.4,
	    "rate_5m": 982.4,
	    "rate_15m": 982.4
    },
    "tags": [
        "metric"
    ]

}
```

这里给metrics event添加了一个metric标签，这样方便与其他业务日志区分开，在后续的ruby处理，email发送邮件，存储ES时，都使用了tag中是否包含metric标签来判断，该日志是否为metrics event。如果是metrics event我们才进行ruby处理，进行event的条件过滤。如果是metrics event我们才发送邮件，并且不保存到ES索引中。

这里我建议使用event.count来作为判断依据，而不是使用rate。因为count更适合用于是判断日志的收集数量，而rate更适合用于判断日志的收集速率。


参考文章：

- https://www.elastic.co/guide/en/logstash/2.3/plugins-filters-metrics.html#plugins-filters-metrics-clear_interval
- https://www.elastic.co/guide/en/logstash/2.3/plugins-outputs-email.html#plugins-outputs-email-address
- https://www.elastic.co/guide/en/logstash/2.3/plugins-filters-ruby.html#plugins-filters-ruby-code
- http://chenlinux.com/2013/07/11/howto-filter-count-in-logstash/
- http://xiaorui.cc/2015/04/16/使用kibana4的新功能metric做数据聚合/
- https://developer.rackspace.com/blog/using-logstash-to-push-metrics-to-graphite/