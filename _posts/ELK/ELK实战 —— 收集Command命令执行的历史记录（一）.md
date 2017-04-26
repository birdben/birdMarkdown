## ELK实战 —— 收集Command命令执行的历史记录（一）

先说明一下我的安装环境，我用的是MacBook所以环境可能和Linux系统稍有不同，这里使用ELK来收集Command命令的执行历史记录，在架构上做了简化直接使用Logstash收集log文件，然后output输出到ES，所以Logstash没有区分Shipper和Indexer，也没有使用Redis做缓冲处理，后续会再细化补充。

### 首先将Command命令保存到log文件中

在/etc/bashrc里添加下面的内容来保存Command命令执行的历史记录到log文件中

```
# HISTDIR是Command命令保存的log文件路径，这里需要注意当前操作用户一定要对该log文件路径有读写权限
HISTDIR='/Users/ben/Downloads/command.log'
if [ ! -f $HISTDIR ];
    then touch $HISTDIR sudo chmod 666 $HISTDIR
fi

# 定义Command日志的格式
export HISTTIMEFORMAT="{\"TIME\":\"%F %T\",\"HOSTNAME\":\"$HOSTNAME\",\"LI\":\"$(who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g')\",\"LU\":\"$(who am i|awk '{print $1}')\",\"NU\":\"${USER}\",\"CMD\":\""

# 输出日志到指定的log文件
export PROMPT_COMMAND='history 1|tail -1|sed "s/^[ ]\+[0-9]\+ //"|sed "s/$/\"}/">> /Users/ben/Downloads/command.log'
```

### Logstash配置文件

```
input {
	file {
		path => ["/Users/ben/Downloads/command.log"]
		type => "command"
		codec => "json"
		start_position => "beginning"
	}
}

output {
    stdout {
        codec=>rubydebug
    }
    elasticsearch {
		embedded => false
		codec => "json"
		host => "127.0.0.1"
		port => 9200
		protocol => "http"
		index => "command_index"
	}
}
```

### ES的mapping需要注意的地方

因为我自己创建的索引command\_index的mapping，所有的String字段都没有指定分词情况，所以ES默认都是Analyzed分词。所以这里只需要将mapping修改一下，将所有的String类型字段都指定分词方式("index":"not\_analyzed")，这样指定的字段就不会分词了，就可以使用Aggregation Terms方式创建图表了。（注意：一定要删除原来的mapping，重新索引数据才可以）

### 重新生成ES索引数据，Logstash配置需要注意的地方start_position和sincedb

因为我们没有做持久化存储，所以当索引数据有问题的时候，我们只能重新从log文件中读取所有日志信息重新生成ES索引

```
# start_position是监听的位置，默认是end，即一个文件如果没有记录它的读取信息，则从文件的末尾开始读取，也就是说，仅仅读取新添加的内容。对于一些更新的日志类型的监听，通常直接使用end就可以了；相反，beginning就会从一个文件的头开始读取。但是如果记录过文件的读取信息，这个配置也就失去作用了。
start_position => "beginning"
		
# sincedb文件使用来保存logstash读取日志文件的进度的
# 默认存储在home路径下.sincedb_c9a33fda01005ad7430b6ef4a0d51f8b，可以设置sincedb_path指定该文件的路径
# c9a33fda01005ad7430b6ef4a0d51f8b是log文件路径"/Users/ben/Downloads/command.log"做MD5后的值
```

### 重新创建索引，@Timestamp的时间是重新创建索引的时间，不是command执行的时间

这里重新创建索引@Timestamp字段是我们重新创建索引的时间，而不是command命令执行的时间，这里我们的日志中是有一个TIME字段是command执行的实际时间的，所以需要在logstash中将TIME字段转成@Timestamp字段，logstash的配置修改如下

```
input {
	file {
		path => ["/Users/ben/Downloads/command.log"]
		type => "command"
		codec => "json"
		start_position => "beginning"
	}
}

filter {
    date {
    	# 将TIME字段的时间输出到@timestamp字段
		match => ["TIME", "yyyy-MM-dd HH:mm:ss"]
		target => "@timestamp"
	}
}

output {
    stdout {
        codec=>rubydebug
    }
    elasticsearch {
		embedded => false
		codec => "json"
		host => "127.0.0.1"
		port => 9200
		protocol => "http"
		index => "command_index"
	}
}
```


### Kibana效果图

![Kibana_Dashboard](http://img.blog.csdn.net/20160815175259042?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

参考文章：

- https://mp.weixin.qq.com/s?__biz=MzIwMDc3Njk4NA==&mid=2247483813&idx=1&sn=e01b82241adfa265b500639057002c73&scene=1&srcid=0803lCZrSkEMZ1PgTHjSNGO5&key=8dcebf9e179c9f3ad1a515afc7035cc46522d515b71ffc2054cd632d93bfefce768b74e094abcb7ff7dae7808e74056a&ascene=0&uin=OTIxOTMxODgw&devicetype=iMac+MacBookPro12查看C1+OSX+OSX+10.10.3+build(14D136)&version=11020201&pass_ticket=74GRaQEgihFnYHPlfay4kIlcDjlfwXktoPgqFOfHSuk4DPLXTLVuxF查看BaA1DdMFdG