## Logstash学习（一）基本用法

### Logstash
简单介绍一下logstash的配置文件由 input filter output 等几个基本的部分组成，顾名思义 input 就是从哪收集数据，output就是输出到哪，filter代表一个过滤规则意思是什么内容会被收集。Logstash基本上用于收集，解析和存储日志。

### Gemfile文件

```
# 指定更新ruby插件的数据源
source "https://ruby.taobao.org/"
gem "logstash-core", "1.5.6"
gem "file-dependencies", "0.1.6"
gem "ci_reporter_rspec", "1.0.0", :group => :development
gem "simplecov", :group => :development
gem "coveralls", :group => :development
gem "tins", "1.6", :group => :development
gem "rspec", "~> 3.1.0", :group => :development
gem "logstash-devutils", "~> 0", :group => :development
gem "benchmark-ips", :group => :development
gem "octokit", "3.8.0", :group => :build
gem "stud", "0.0.21", :group => :build
gem "fpm", "~> 1.3.3", :group => :build
gem "rubyzip", "~> 1.1.7", :group => :build
gem "gems", "~> 0.8.3", :group => :build
gem "flores", "~> 0.0.6", :group => :development
gem "logstash-input-heartbeat"
gem "logstash-output-zeromq"
gem "logstash-codec-collectd"
gem "logstash-output-xmpp"
gem "logstash-codec-dots"
...
gem "logstash-input-beats"
```
### 下面主要列出了一些常用的插件

### Input Plugin
#### 

下面列举出了常用的input插件，*开头的是logstash_1.5默认安装的插件

```
*logstash-input-beats
*logstash-input-file
*logstash-input-http
 logstash-input-jdbc
 logstash-input-jmx
*logstash-input-kafka
*logstash-input-log4j
*logstash-input-rabbitmq
*logstash-input-redis
*logstash-input-stdin
 logstash-input-sqlite
*logstash-input-syslog
*logstash-input-tcp
 logstash-input-websocket
```

### Codec Plugin

下面列举出了常用的codec插件，*开头的是logstash_1.5默认安装的插件

```
*logstash-codec-collectd
*logstash-codec-json_lines
*logstash-codec-json
*logstash-codec-line
*logstash-codec-multiline
*logstash-codec-plain
*logstash-codec-rubydebug
```

### Filter Plugin

下面列举出了常用的filter插件，*开头的是logstash_1.5默认安装的插件

```
*logstash-filter-date
*logstash-filter-drop
*logstash-filter-geoip
*logstash-filter-grok
 logstash-filter-i18n
*logstash-filter-json
 logstash-filter-json_encode
*logstash-filter-kv
*logstash-filter-mutate
*logstash-filter-metrics
*logstash-filter-multiline
*logstash-filter-ruby
 logstash-filter-range
*logstash-filter-split
*logstash-filter-uuid
*logstash-filter-xml
```

### Output Plugin

下面列举出了常用的output插件，*开头的是logstash_1.5默认安装的插件

```
*logstash-output-elasticsearch
*logstash-output-file
*logstash-output-http
*logstash-output-kafka
 logstash-output-mongodb
*logstash-output-rabbitmq
*logstash-output-redis
 logstash-output-solr_http
 logstash-output-syslog
*logstash-output-stdout
*logstash-output-tcp
 logstash-output-websocket
 logstash-output-zabbix
*logstash-output-zeromq
```



## logstash-input-file插件的用法

#### start_position用法

```
# start_position是监听的位置，默认是end，即一个文件如果没有记录它的读取信息，则从文件的末尾开始读取，也就是说，仅仅读取新添加的内容。对于一些更新的日志类型的监听，通常直接使用end就可以了；相反，beginning就会从一个文件的头开始读取。但是如果记录过文件的读取信息，这个配置也就失去作用了。
start_position => "beginning"
```

#### sincedb用法

```
# sincedb文件使用来保存logstash读取日志文件的进度的
# 默认存储在home路径下.sincedb_c9a33fda01005ad7430b6ef4a0d51f8b，可以设置sincedb_path指定该文件的路径
# c9a33fda01005ad7430b6ef4a0d51f8b是log文件路径"/Users/ben/Downloads/command.log"做MD5后的值
```



## logstash-filter-grok插件的用法

#### grok使用自定义正则表达式

#### ${LOGSTASH_HOME}/patterns/postfix

```
BDP_LOGMESSAGE %{DATA:logInfo.startTimestamp}\|%{DATA:logInfo.endTimestamp}\|%{INT:logInfo.cost}\|%{DATA:logInfo.userID}\|%{DATA:logInfo.userName}\|%{DATA:logInfo.departmentName}\|%{DATA:logInfo.module}\|%{DATA:logInfo.function}\|%{DATA:logInfo.op}\|%{DATA:logInfo.status}\|%{DATA:logInfo.message}\|%{DATA:logInfo.target}\|%{DATA:logInfo.targetDetail}\|
```

#### logstash.conf文件中的filter需要指定自定义grok表达式的文件路径


```
filter {
	# 指定自定义grok正则表达式文件的路径
	patterns_dir => "./patterns"
	# 使用了自定义的BDP_LOGMESSAGE表达式去匹配message字段，将message中匹配BDP_LOGMESSAG表达式的内容拆分成指定的字段
	grok {
		match => {
			"message" => "%{TIMESTAMP_ISO8601:date} %{LOGLEVEL:level}  \[%{WORD:priotiy}\] \- %{BDP_LOGMESSAGE}"
		}
	}
}
```


```
filter {
  if [type] == "tomcatlog" {
     multiline {
       pattern => "^%{TIMESTAMP_ISO8601}"
       negate => true
       what=> "previous"
     }
     if "_grokparsefailure" in [tags] {
       drop { }
     }
     grok {
       match => { "message" =>
         "%{TIMESTAMP_ISO8601:date} \[(?<thread_name>.+?)\] (?<log_level>\w+)\s*(?<content>.*)"
            }
     } 
     
     date {
       match => [ "timestamp", "yyyy-MM-dd HH:mm:ss,SSS Z", "MMM dd, yyyy HH:mm:ss a" ]
     }
  }
}
```