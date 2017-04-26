## ElasticSearch环境搭建（2.x版本）

基本步骤如下（ES版本是2.3.x）

1. 安装Java环境
2. 安装ES
3. 安装head，bigdesk插件
4. 安装ik分词插件
5. 安装jdbc-river插件（后续补上）
6. 其他插件的安装（后续补上）

### 安装Java环境

```
# 下载安装JDK8，具体JDK的下载地址不提供了
$ tar -zxvf jdk-8u91-linux-x64.tar.gz
$ mkdir /usr/local/java/
$ cp -r jdk1.8.0_91 /usr/local/java/

# 配置Java环境变量
$ vi /etc/profile

# 添加Java环境变量
export JAVA_HOME=/usr/local/java/jdk1.8.0_91
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export CLASSPATH=$CLASSPATH:.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
```

### 安装ES环境
```
# 下载安装ES 2.3.5
$ wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-2.3.5.tar.gz
$ tar -zxvf elasticsearch-2.3.5.tar.gz
$ mkdir /opt/
$ cp -r elasticsearch-2.3.5 /opt/

# 启动ES
$ cd /opt/elasticsearch-2.3.5/bin
$ ./elasticsearch

# 如果启动的时候报错，是没有权限创建log文件，尝试给ES_HOME目录的权限授予给当前用户
$ chown -R ben:root /opt/elasticsearch-2.3.5
```

### 安装ES插件head
```
# 两种安装方式（在线安装和离线安装）
# 注意：2.x和1.x版本的安装可能稍有不同，具体参考head的github

# 在线安装head插件
$ cd /opt/elasticsearch-2.3.5/bin
$ ./plugin install mobz/elasticsearch-head

# 离线安装head插件
# 从github下载head插件的master.zip
$ cd /opt/elasticsearch-2.3.5/bin
$ ./plugin --install head --url file:///home/ben/Downloads/elasticsearch-head-master.zip

# 具体参考github地址：
https://github.com/mobz/elasticsearch-head
```

### 安装ik插件
```
# 手动安装ik插件和1.x版本没什么区别，从github下载对应的ik插件版本，然后maven手动编译打包
$ git clone https://github.com/medcl/elasticsearch-analysis-ik
$ cd elasticsearch-analysis-ik
$ mvn clean
$ mvn compile
$ mvn package

# 拷贝和解压release下的文件: #{project_path}/elasticsearch-analysis-ik-*/target/releases/elasticsearch-analysis-ik-*.zip 到你的 elasticsearch 插件目录, 如: plugins/ik 重启elasticsearch
# 这里2.x版本直接把所有的配置都放在zip包里了，1.x版本还需要自己手动复制config配置文件夹，现在直接解压zip包到${ES_HOME}/plugins/ik/目录即可
$ cd #{project_path}/elasticsearch-analysis-ik-*/target/releases
$ unzip -o elasticsearch-analysis-ik-*.zip -d ${ES_HOME}/plugins/ik/

# 注意：2.x的版本安装ik插件不需要修改elasticsearch.yml配置文件了

# 具体参考github地址：
https://github.com/medcl/elasticsearch-analysis-ik
```


如果报错如下，是因为没有将elasticsearch-analysis-ik-*.zip的内容解压到${ES_HOME}/plugins/ik/目录

```
Exception in thread "main" java.lang.IllegalStateException: Could not load plugin descriptor for existing plugin [analysis-ik]. Was the plugin built before 2.0?
Likely root cause: java.nio.file.NoSuchFileException: /home/es/es2/plugins/analysis-ik/plugin-descriptor.properties
```

### 访问地址

如果需要通过ip进行访问es集群，必须修改elasticsearch.yml中的network.host节点。es 1.x版本的默认配置是"0.0.0.0"，所以不绑定ip也可访问，但是es 2.x版本如果采用默认配置，只能通过 localhost 和 "127.0.0.1"进行访问，所以这里需要修改network.host配置成内网IP或者外网IP。network.host也可以配置多个值。

```
# 该IP根据自己环境的实际情况修改
network.host: 10.23.41.252
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
            "analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "age": {
            "type": "integer"
          },
          "job": {
            "type": "string",
            "analyzer": "ik",
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
            "analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "age": {
            "type": "integer"
          },
          "job": {
            "type": "string",
            "analyzer": "ik",
            "search_analyzer": "ik_smart"
          },
          "createTime": {
            "type": "long"
          }
        }
      }
    }
}'

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
