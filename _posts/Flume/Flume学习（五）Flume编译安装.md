---
title: "Flume学习（五）Flume编译安装"
date: 2016-08-26 13:07:11
tags: [Flume]
categories: [Log]
---

### 下载Flume源码

```
$ curl -o apache-flume-1.6.0-src.tar.gz http://mirrors.hust.edu.cn/apache/flume/1.6.0/apache-flume-1.6.0-src.tar.gz
$ tar -zxvf apache-flume-1.6.0-src.tar.gz
$ cd apache-flume-1.6.0-src
```

### 使用Maven编辑打包

```
$ mvn clean
$ mvn install -DskipTests -Dtar
```

编译过程中，下载ua-parser-1.3.0.pom可能会失败，出现类似下面的错误，无法连接到http://maven.twttr.com:80，会重复尝试几次连接不上就build failed了。这个可能是因为http://maven.twttr.com:80地址被墙了

```
[INFO] ------------------------------------------------------------------------
[INFO] Building Flume NG Morphline Solr Sink 1.6.0
[INFO] ------------------------------------------------------------------------
Downloading: http://10.10.1.10:8082/content/groups/public/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
Downloading: https://repository.cloudera.com/artifactory/cloudera-repos/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
Downloading: http://repo1.maven.org/maven2/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
Downloading: http://repository.jboss.org/nexus/content/groups/public/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
Downloading: https://repo.maven.apache.org/maven2/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
Downloading: http://maven.twttr.com/ua_parser/ua-parser/1.3.0/ua-parser-1.3.0.pom
八月 26, 2016 12:11:25 下午 org.apache.maven.wagon.providers.http.httpclient.impl.execchain.RetryExec execute
信息: I/O exception (java.net.NoRouteToHostException) caught when processing request to {}->http://maven.twttr.com:80: No route to host
八月 26, 2016 12:11:25 下午 org.apache.maven.wagon.providers.http.httpclient.impl.execchain.RetryExec execute
信息: Retrying request to {}->http://maven.twttr.com:80
```

我把网上解决无法连接http://maven.twttr.com的办法汇总了一下，需要在Flume的pom文件中最上面添加下面这些repository地址，来下载ua-parser-1.3.0.pom

```
<repositories>
	<repository>
	  <id>maven.tempo-db.com</id>
	  <url>http://maven.oschina.net/service/local/repositories/sonatype-public-grid/content/</url>
	</repository>
	<repository>
	  <id>p2.jfrog.org</id>
	  <url>http://p2.jfrog.org/libs-releases</url>
	</repository>
	<repository>
	  <id>nexus.axiomalaska.com</id>
	  <url>http://nexus.axiomalaska.com/nexus/content/repositories/public</url>
	</repository>
</repositories>
```

添加后如下图所示

![Maven](http://img.blog.csdn.net/20160826121945067?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

再次编译打包后，就成功了。可以在apache-flume-1.6.0-src/flume-ng-dist/target/路径下找到我们打包好的apache-flume-1.6.0-bin.tar.gz，这样我们就可以在flume的源代码上进行自己的修改并且可以打包应用了。

```
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] Apache Flume ....................................... SUCCESS [  0.928 s]
[INFO] Flume NG SDK ....................................... SUCCESS [  3.490 s]
[INFO] Flume NG Configuration ............................. SUCCESS [  1.329 s]
[INFO] Flume Auth ......................................... SUCCESS [  4.949 s]
[INFO] Flume NG Core ...................................... SUCCESS [  8.924 s]
[INFO] Flume NG Sinks ..................................... SUCCESS [  0.050 s]
[INFO] Flume NG HDFS Sink ................................. SUCCESS [  2.947 s]
[INFO] Flume NG IRC Sink .................................. SUCCESS [  1.002 s]
[INFO] Flume NG Channels .................................. SUCCESS [  0.041 s]
[INFO] Flume NG JDBC channel .............................. SUCCESS [  1.691 s]
[INFO] Flume NG file-based channel ........................ SUCCESS [  6.320 s]
[INFO] Flume NG Spillable Memory channel .................. SUCCESS [  1.323 s]
[INFO] Flume NG Node ...................................... SUCCESS [  2.382 s]
[INFO] Flume NG Embedded Agent ............................ SUCCESS [  1.458 s]
[INFO] Flume NG HBase Sink ................................ SUCCESS [  3.763 s]
[INFO] Flume NG ElasticSearch Sink ........................ SUCCESS [  1.914 s]
[INFO] Flume NG Morphline Solr Sink ....................... SUCCESS [02:41 min]
[INFO] Flume Kafka Sink ................................... SUCCESS [  1.194 s]
[INFO] Flume NG Kite Dataset Sink ......................... SUCCESS [  2.804 s]
[INFO] Flume NG Hive Sink ................................. SUCCESS [  2.462 s]
[INFO] Flume Sources ...................................... SUCCESS [  0.030 s]
[INFO] Flume Scribe Source ................................ SUCCESS [  1.081 s]
[INFO] Flume JMS Source ................................... SUCCESS [  1.362 s]
[INFO] Flume Twitter Source ............................... SUCCESS [  0.895 s]
[INFO] Flume Kafka Source ................................. SUCCESS [  1.066 s]
[INFO] flume-kafka-channel ................................ SUCCESS [  1.093 s]
[INFO] Flume legacy Sources ............................... SUCCESS [  0.028 s]
[INFO] Flume legacy Avro source ........................... SUCCESS [  0.998 s]
[INFO] Flume legacy Thrift Source ......................... SUCCESS [  1.248 s]
[INFO] Flume NG Clients ................................... SUCCESS [  0.026 s]
[INFO] Flume NG Log4j Appender ............................ SUCCESS [  3.208 s]
[INFO] Flume NG Tools ..................................... SUCCESS [  0.902 s]
[INFO] Flume NG distribution .............................. SUCCESS [ 12.454 s]
[INFO] Flume NG Integration Tests ......................... SUCCESS [  1.278 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 03:57 min
[INFO] Finished at: 2016-08-26T12:24:44+08:00
[INFO] Final Memory: 411M/868M
[INFO] ------------------------------------------------------------------------
```

参考文章：

- http://blog.csdn.net/yydcj/article/details/38824823
- https://www.iteblog.com/archives/1043
- http://blog.csdn.net/qianshangding0708/article/details/48087911
- http://blog.csdn.net/yeruby/article/details/50751927