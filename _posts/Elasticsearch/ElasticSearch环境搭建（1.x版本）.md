## ElasticSearch环境搭建（1.x版本）

基本步骤如下（ES版本是1.7.x）

1. 安装Java环境
2. 安装ES
3. 安装head，bigdesk插件
4. 安装ik分词插件
5. 安装jdbc-river插件
6. 使用ESClient测试

### 安装Java环境

```
# 下载安装JDK7，具体JDK的下载地址不提供了
$ tar -zxvf jdk-7u71-linux-x64.tar.gz
$ mkdir /usr/local/java/
$ cp -r jdk1.7.0_67 /usr/local/java/

# 配置Java环境变量
$ vi /etc/profile

# 添加Java环境变量
export JAVA_HOME=/usr/local/java/jdk1.7.0_67
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export CLASSPATH=$CLASSPATH:.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
```

### 安装ES环境
```
# 下载安装ES 1.7.3
$ wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.3.tar.gz
$ tar -zxvf elasticsearch-1.7.3.tar.gz
$ mkdir /opt/
$ cp -r elasticsearch-1.7.3 /opt/

# 启动ES
$ cd /opt/elasticsearch-1.7.3/bin
$ ./elasticsearch

# 如果启动的时候报错，是没有权限创建log文件，尝试给ES_HOME目录的权限授予给当前用户
$ chown -R ben:root /opt/elasticsearch-1.7.3
```

### 安装ES插件head
```
# 两种安装方式（在线安装和离线安装）
# 注意：2.x和1.x版本的安装可能稍有不同，具体参考head的github

# 在线安装head插件
$ cd /opt/elasticsearch-1.7.3/bin
$ ./plugin --install mobz/elasticsearch-head

# 离线安装head插件
# 从github下载head插件的master.zip
$ cd /opt/elasticsearch-1.7.3/bin
$ ./plugin --install head --url file:///home/ben/Downloads/elasticsearch-head-master.zip
```

### 安装ik插件
```
# 从github下载ik插件，因为我使用的是老版本的ES，所以ik插件也使用的是对应老版本的
$ git clone https://github.com/medcl/elasticsearch-analysis-ik
$ cd elasticsearch-analysis-ik
$ mvn clean
$ mvn compile
$ mvn package
$ cd target
$ cp elasticsearch-analysis-ik-xxx.jar ${ES_HOME}/plugins/ik/

$ cd elasticsearch-analysis-ik
$ cp config/ik ${ES_HOME}/config/

# elasticsearch.yml配置文件中添加ik分词配置
index:
  analysis:
    analyzer:
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true

index.analysis.analyzer.default.type: ik
index.store.type: niofs
```

### 测试ik分词

```
# User的mapping
{
    "mappings": {
      "user": {
        "dynamic" : "strict",
        "properties": {
          "id": {
            "type": "string",
            "index": "not_analyzed"
          },
          "name": {
            "type": "string",
            "index_analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "age": {
            "type": "integer"
          },
          "job": {
            "type": "string",
            "index_analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "createTime": {
            "type": "long"
          }
        }
      }
    }
}

# 尝试创建user索引
$ curl -XPOST 'http://127.0.0.1:9200/user?pretty' -d '{
    "mappings": {
      "user": {
        "dynamic" : "strict",
        "properties": {
          "id": {
            "type": "string",
            "index": "not_analyzed"
          },
          "name": {
            "type": "string",
            "index_analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "age": {
            "type": "integer"
          },
          "job": {
            "type": "string",
            "index_analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "createTime": {
            "type": "long"
          }
        }
      }
    }
}'

# 注意执行curl命令时，如果遇到下面的错误，原因是${ES_HOME}/lib/下需要引入httpclient-4.5.jar, httpcore-4.4.1.jar
{
  "error" : "IndexCreationException[[user] failed to create index]; nested: NoClassDefFoundError[org/apache/http/client/ClientProtocolException]; nested: ClassNotFoundException[org.apache.http.client.ClientProtocolException]; ",
  "status" : 500
}

# 创建索引成功后，查看索引信息
$ curl -XGET 'http://127.0.0.1:9200/_cat/indices?pretty'
green open user 5 1 0 0 970b 575b

# 测试standard分词效果
$ curl -XGET 'http://127.0.0.1:9200/user/_analyze?analyzer= standard&pretty=true' -d '{"text":"中华人民共和国国歌"}'

# 测试ik分词效果
$ curl -XGET 'http://127.0.0.1:9200/user/_analyze?analyzer=ik&pretty=true' -d '{"text":"中华人民共和国国歌"}'

# 测试ik_smart分词效果
$ curl -XGET 'http://127.0.0.1:9200/user/_analyze?analyzer=ik_smart&pretty=true' -d '{"text":"中华人民共和国国歌"}'
```

### 安装jdbc-river插件
```
# 下载安装jdbc-river
# 由于es官网叫停river类的导入插件，因此原始的elasticsearch-jdbc-river变更为elasticsearch-jdbc，成为一个独立的导入工具。官方提到的同类型工具还有logstash。
# 注意jdbc-river在1.5之前的版本和之后的版本有很大的区别，版本对应关系请参考：https://github.com/jprante/elasticsearch-jdbc
# 这里安装的1.7.3.0版本对应ES的1.7.3版本

# 安装方式也有所变化，1.5.0版本之后只需要直接下载zip包解压即可，以前的命令安装方式已经不再适用，所以不要使用下面的命令安装了
./bin/plugin --install river-jdbc --url  http://xbib.org/repository/org/xbib/elasticsearch/plugin/elasticsearch-river-jdbc/1.5.0.5/elasticsearch-river-jdbc-1.5.0.5-plugin.zip

# 1.5版本之后改为jdbc-importer，具体的导入配置区别请参考：https://github.com/jprante/elasticsearch-jdbc/wiki/JDBC-plugin-feeder-mode-as-an-alternative-to-the-deprecated-Elasticsearch-River-API

$ wget http://xbib.org/repository/org/xbib/elasticsearch/importer/elasticsearch-jdbc/1.7.3.0/elasticsearch-jdbc-1.7.3.0-dist.zip
$ unzip elasticsearch-jdbc-1.7.3.0-dist.zip
$ cp -r elasticsearch-jdbc-1.7.3.0 /opt/

$ cd /opt/elasticsearch-1.7.3/bin
$ mkdir jdbc_importer
$ cd jdbc_importer

# 注意：执行该sh脚本的时候，需要给root账号开启远程访问权限
# 编辑mysql_user.sh脚本，用来全量同步MySQL的数据到ES
$ vi mysql_user.sh
# 编辑mysql_user_schedule.sh脚本，用来增量同步MySQL的数据到ES
$ vi mysql_user_schedule.sh
```


##### mysql_user.sh
```
#!/bin/bash

bin=/opt/elasticsearch-jdbc-1.7.3.0/bin
lib=/opt/elasticsearch-jdbc-1.7.3.0/lib

echo '
{
"type" : "jdbc",
"jdbc" : {
    "url" : "jdbc:mysql://127.0.0.1:3306/test",
    "user" : "root",
    "password" : "root",
    "sql" :  "select t.id as _id, t.id as id, t.name as name, t.age as age, t.job as job, t.createTime as createTime from user t",
    "elasticsearch.cluster" : "es",
    "elasticsearch.host" : "127.0.0.1:9300",
    "index" : "user",
    "type" : "user"
  }
}
' | java \
-cp "${lib}/*" \
-Dlog4j.configurationFile="${bin}/log4j2.xml" \
"org.xbib.tools.Runner" \
"org.xbib.tools.JDBCImporter"

```


##### mysql_user_schedule.sh
```
#!/bin/bash

bin=/opt/elasticsearch-jdbc-1.7.3.0/bin
lib=/opt/elasticsearch-jdbc-1.7.3.0/lib

echo '
{
"type" : "jdbc",
"jdbc" : {
    "url" : "jdbc:mysql://127.0.0.1:3306/test",
    "user" : "root",
    "password" : "root",
    "schedule" : "0 0/1 * * * ?",
    "interval" : 1,
    "sql" : [
                { 
                     "statement" : "select t.id as _id, t.id as id, t.name as name, t.age as age, t.job as job, t.createTime as createTime from user t where t.createTime > ?", 
                     "parameter" : [ "$metrics.lastexecutionstart" ]
                }
    ],
    "elasticsearch.cluster" : "es",
    "elasticsearch.host" : "127.0.0.1:9300",
    "index" : "user",
    "type" : "user"
  }
}
' | java \
-cp "${lib}/*" \
-Dlog4j.configurationFile="${bin}/log4j2.xml" \
"org.xbib.tools.Runner" \
"org.xbib.tools.JDBCImporter"

```

### 使用ESClient测试
后续完善

