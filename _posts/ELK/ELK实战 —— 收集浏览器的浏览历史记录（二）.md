---
title: "ELK实战 —— 收集浏览器的浏览历史记录（二）"
date: 2016-08-15 23:25:13
tags: [ELK]
categories: [Log]
---

### 这里我们只收集Safari，Chrome，FireFox三个主流浏览器

#### 首先需要找到各个浏览器的历史记录所存储在本地操作系统的路径
所有浏览器的历史记录的存储都是用的sqlite数据库，是一个轻量级的客户端数据库，支持标准的sql查询，可以使用sqlite数据库客户端连接（我这里用的Navicat）。我用的Mac操作系统，所以我的sqlite数据库文件的地址如下：（不同的操作系统路径有所不同，请根据自身环境查找）

```
# Safari
/Users/ben/Library/Safari/History.db
# Chrome
/Users/ben/Library/Application Support/Google/Chrome/Default/History
# FireFox
/Users/ben/Library/Application Support/Firefox/Profiles/teofen8x.default/places.sqlite
```

Safari浏览器数据库

![Safari浏览器数据库](http://img.blog.csdn.net/20160808111511672?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Chrome浏览器数据库

![Chrome浏览器数据库](http://img.blog.csdn.net/20160808155407372?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

FireFox浏览器数据库

![FireFox浏览器数据库](http://img.blog.csdn.net/20160808111741313?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 打开sqlite数据库，遇到'database is locked'问题
原因是sqlite不支持并发执行写入操作，即使是不同的表，只支持库级锁。所以Navicat打开sqlite数据库文件时会提示'database is locked'。知道了sqlite报这个错误的原因了，解决起来也比较简单，只要关掉当前浏览器的所有进程，重新打开sqlite文件就不会报错了，因为没有浏览器进程访问sqlite数据库了。

#### Logstash解析sqlite遇到Cannot find Serializer for class: org.jruby.RubyObject错误
```
{:timestamp=>"2016-08-06T11:45:25.875000+0800", :message=>"Got error to send bulk of actions: Cannot find Serializer for class: org.jruby.RubyObject", :level=>:error}
{:timestamp=>"2016-08-06T11:45:25.875000+0800", :message=>"Failed to flush outgoing items", :outgoing_count=>80, :exception=>"JrJackson::ParseError", :backtrace=>["com/jrjackson/JrJacksonBase.java:78:in `generate'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/jrjackson-0.3.7/lib/jrjackson/jrjackson.rb:59:in `dump'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/multi_json-1.11.2/lib/multi_json/adapters/jr_jackson.rb:20:in `dump'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/multi_json-1.11.2/lib/multi_json/adapter.rb:25:in `dump'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/multi_json-1.11.2/lib/multi_json.rb:136:in `dump'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/elasticsearch-api-1.0.15/lib/elasticsearch/api/utils.rb:102:in `__bulkify'", "org/jruby/RubyArray.java:2414:in `map'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/elasticsearch-api-1.0.15/lib/elasticsearch/api/utils.rb:102:in `__bulkify'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/elasticsearch-api-1.0.15/lib/elasticsearch/api/actions/bulk.rb:82:in `bulk'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/logstash-output-elasticsearch-1.1.0-java/lib/logstash/outputs/elasticsearch/protocol.rb:105:in `bulk'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/logstash-output-elasticsearch-1.1.0-java/lib/logstash/outputs/elasticsearch.rb:548:in `submit'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/logstash-output-elasticsearch-1.1.0-java/lib/logstash/outputs/elasticsearch.rb:572:in `flush'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/stud-0.0.21/lib/stud/buffer.rb:219:in `buffer_flush'", "org/jruby/RubyHash.java:1342:in `each'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/stud-0.0.21/lib/stud/buffer.rb:216:in `buffer_flush'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/stud-0.0.21/lib/stud/buffer.rb:193:in `buffer_flush'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/stud-0.0.21/lib/stud/buffer.rb:112:in `buffer_initialize'", "org/jruby/RubyKernel.java:1479:in `loop'", "/Users/ben/dev/logstash-1.5.6/vendor/bundle/jruby/1.9/gems/stud-0.0.21/lib/stud/buffer.rb:110:in `buffer_initialize'"], :level=>:warn}
```

刚好在github的issue里面有提到过，后来自己看了一下[#298](https://github.com/logstash-plugins/logstash-output-elasticsearch/issues/298)这个issue，从中找到了问题的原因。问题的原因就是sqlite的数据中某个字段无法用JrJackson序列化，也就是数据有问题，后来我仔细观察了一下我自己的sqlite数据，果然有个db字段的数据类型有问题，我在logstash filter中把db这个字段remove了就不会报错了。

```
{
                    "host" => "localhost",
                      "db" => #<Sequel::JDBC::Database: "jdbc:sqlite:/Users/ben/Library/Safari/History.db">,
                "@version" => "1",
              "@timestamp" => "2016-08-08T05:54:37.985Z",
                    "type" => "safari_history_visits",
            "history_item" => 55723,
              "visit_time" => 492328032.0,
                   "title" => "Cannot find Serializer for class: org.jruby.RubyObject · Issue #298 · logstash-plugins/logstash-output-elasticsearch",
         "load_successful" => true,
            "http_non_get" => false,
             "synthesized" => false,
         "redirect_source" => nil,
    "redirect_destination" => nil,
                  "origin" => 0,
              "generation" => 31154
}
```

参考文章：

- https://github.com/logstash-plugins/logstash-output-elasticsearch/issues/298


#### 不同浏览器的timestamp时间格式问题

每个浏览器的timestamp时间戳字段用的格式都不一样，真的很头疼啊

##### Safari Timestamp
Safari的浏览时间是history\_visit表中的visit\_time字段，"visit\_time" => 492328032.0，根据这个时间戳转换出来的实际时间是'1985/8/8 13:47:12'（浏览时间是1985年老子还没出生呢，根本不可能啊），后来又google了一下，发现Safari用的NSDate，是IOS的一个日期类型，这个时间戳是从2001-01-01 00:00:00开始的，所以要加上492328032.0 (visit\_time) + 978307200 = 1470635232转换出来的实际时间'2016-08-08 13:47:12'才正确。

```
# NSDate Class Reference
NSDate objects encapsulate a single point in time, independent of any particular calendrical system or time zone. Date objects are immutable, representing an invariant time interval relative to an absolute reference date (00:00:00 UTC on 1 January 2001).
```

##### FireFox Timestamp
FireFox的浏览时间是moz\_historyvisits表中的visit\_date字段，"visit\_date" => 1470469974041830，但是这个时间戳既不是10位的也不是13位的，所以截取前13位就能转换成正确的实际时间'2016-08-06 15:52:54'

##### Chrome Timestamp
在visit_time和last_visit_time字段下，我的是13115139002559995。这么长的一串字肯定是微秒级的（真TM长啊。。），按照Unix时间戳截断10位后1311513900，转换成实际的时间是'2011/7/24 21:25:0'这也不对啊，最后一次使用Chrome浏览器浏览网页明明是昨天啊。。我又郁闷了。。还是找了google老大帮忙，发现Chrome的浏览记录visit_time和last_visit_time的起始值是1601年1月1日0时0分0秒，和Safari一样坑。。所以要减去11644473600 (visit\_time) - 11644473600 = 1470665402转换出来的实际时间'2016-08-08 22:10:02'才正确

```
"[Google Chrome's] timestamp is formatted as the number of microseconds since January, 1601"

> SELECT datetime((visit_time/1000000)-11644473600, 'unixepoch', 'localtime') AS time FROM visits ORDER BY visit_time DESC LIMIT 10;
```

参考文章：

- http://stackoverflow.com/questions/34167003/what-format-is-the-safari-history-db-history-visits-visit-time-in
- https://groups.google.com/a/chromium.org/forum/#!topic/chromium-discuss/YPMscYNqZFk
- http://stackoverflow.com/questions/20458406/what-is-the-format-of-chromes-timestamps
- http://linuxsleuthing.blogspot.in/2011/06/decoding-google-chrome-timestamps-in.html

#### Logstash启动后没有debug日志输出，Logstash进程一直是sleeping状态
![Logstash进程状态](http://img.blog.csdn.net/20160809012237279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
后来发现是因为Logstash的filter插件配置有问题，导致没有匹配到相应的日志，所以进程一直处于sleeping等待状态


#### 无法同步全量的浏览到ES服务器
启动Logstash同步一次sqlite数据库之后，发现以后再重启Logstash服务也无法重新再读取全部的sqlite数据，后来看了一下logstash-input-sqlite插件的源码，发现logstash-input-sqlite插件会创建一个表since_table用来记录读取sqlite数据库的表和对应表的last_id，这样即使重启Logstash服务，logstash-input-sqlite插件也会从since_table表中记录的最后一次读取的id开始读取，所以如果想要全量重新同步sqlite中的数据就把since_table这个表删了

#### logstash无法处理时间戳转换问题
（未完待续）
logstash的date,mutate,grok插件都不支持上面所描述的浏览器时间戳做加减操作，所以只能扩展logstash-input-sqlite插件，在sqlite插件中做sql查询来处理，但是查看了logstash-input-sqlite插件的源码后，发现logstash-input-sqlite插件都是用的select *方式做的表查询，在logstash的配置中也无法指定相应的字段，所以需要对logstash-input-sqlite插件本身进行扩展。

#### grok使用自定义正则表达式

Safari和Chrome的时间戳转换的问题可能在Logstash是无法操作的，但是FireFox的时间戳只是需要做简单的截取就可以转换成实际的时间，这样我们只需要自己写一个grok正则表达式来匹配上FireFox的时间戳就可以了。

自定义的grok正则表达式我们是存储在${LOGSTASH_HOME}/patterns/postfix文件中的，这里我们是截取了FireFox时间戳的前13位

#### ${LOGSTASH_HOME}/patterns/postfix

```
FIREFOX_TIME [0-9]{13}
```

#### logstash_firefox.conf
```
...

filter {
	grok {
		# 指定自定义grok正则表达式文件的路径
		patterns_dir => "./patterns"
		# 使用了自定义的FIREFOX_TIME表达式去匹配last_visit_date字段
		match => {
           "last_visit_date" => "%{FIREFOX_TIME:last_visit_date}"
        }
        overwrite => [ "last_visit_date" ]
	}
	mutate {
		remove_field => [ "db" ]
    }
    date {
		match => ["last_visit_date", "UNIX_MS"]
		target => "@timestamp"
	}
}

...
```

#### logstash-input-sqlite插件不支持sqlite做多表关联查询操作
（未完待续）
还有就是这三个浏览器的url地址和visitTime浏览时间是分开存储的，url地址都是放在统计表中的，而visitHistory浏览历史只记录url的id和浏览时间，所以要想提取原始的visit浏览数据需要做sqlite表关联查询，所以这里也只能扩展logstash-input-sqlite插件的功能。

```
FireFox : moz_places, moz_historyvisits
Chrome : urls, visits
Safari : history_items, history_visits
```


### （未完待续）
