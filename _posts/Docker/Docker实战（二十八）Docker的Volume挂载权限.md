---
title: "Docker实战（二十八）Docker的Volume挂载权限"
date: 2017-05-02 21:46:56
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

最近在重构Docker镜像的时候，遇到了Volume挂载文件的权限问题。这里测试使用的是Elasticsearch官方提供的镜像。

Elasticsearch官方的Dockerfile文件

- https://github.com/docker-library/elasticsearch/blob/35d99e915d909688807c507a59a2c06039ac92b2/5/Dockerfile
- https://github.com/docker-library/elasticsearch/blob/35d99e915d909688807c507a59a2c06039ac92b2/5/docker-entrypoint.sh

在制作自己的ES镜像的时候，参考了Elasticsearch官方的Dockerfile，有个地方没有弄明白，为什么Dockerfile和docker-entrypoint都要去chown下面的两个目录，在Dockerfile执行一次chown不就可以了吗？不执行chown会有什么问题呢？。

```
chown -R elasticsearch:elasticsearch: /usr/share/elasticsearch/data
chown -R elasticsearch:elasticsearch: /usr/share/elasticsearch/logs
```

带着上面的疑问，我开始做了下面的尝试，这里我在本地使用修改后的Elasticsearch官方的Dockerfile开始构建Docker镜像，然后和官方pull下来的镜像做对比。

先下载Elasticsearch官方的Docker镜像

```
# 下载Elasticsearch官方的Docker镜像
$ docker pull elasticsearch:5.3.1

# 运行Elasticsearch的Docker容器，并且挂载对应的data目录
$ docker run -d -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/official_5/data:/usr/share/elasticsearch/data --name elasticsearch_official_5x elasticsearch:5.3.1

# 进入Docker容器
$ docker exec -it elasticsearch_official_5x /bin/bash

# 查看Docker容器内/usr/share/elasticsearch目录的权限
$ ls -lh /usr/share/elasticsearch
total 228K
-rw-r--r--  1 root          root          190K Apr 17 15:55 NOTICE.txt
-rw-r--r--  1 root          root          9.4K Apr 17 15:55 README.textile
drwxr-xr-x  2 root          root          4.0K Apr 27 00:01 bin
drwxr-xr-x  1 elasticsearch elasticsearch 4.0K Apr 27 00:01 config
drwxr-xr-x  3 elasticsearch elasticsearch  102 May  4 06:20 data
drwxr-xr-x  2 root          root          4.0K Apr 27 00:01 lib
drwxr-xr-x  1 elasticsearch elasticsearch 4.0K Apr 27 00:01 logs
drwxr-xr-x 12 root          root          4.0K Apr 27 00:01 modules
drwxr-xr-x  2 root          root          4.0K Apr 17 15:55 plugins

# 查看Docker容器内/usr/share/elasticsearch/data目录的权限
$ ls -lh /usr/share/elasticsearch/data/
total 0
drwxr-xr-x 3 root root 102 May  4 06:20 nodes

# 查看Elasticsearch进程
$ ps -ef | grep elasticsearch
elastic+     1     0 16 06:19 ?        00:00:17 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms2g -Xmx2g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/usr/share/elasticsearch -cp /usr/share/elasticsearch/lib/elasticsearch-5.3.1.jar:/usr/share/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
root       105    94  0 06:21 ?        00:00:00 grep elasticsearch
```

使用Elasticsearch官方的Docker容器，又发现新的问题，因为Dockerfile中使用了chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data将该目录所有者修改为elasticsearch用户了，为什么/usr/share/elasticsearch/data目录下的文件和文件夹确实属于root用户呢？这个问题暂时先放一边，后面会给出解释，我们先继续之前的尝试。

注意：下面的尝试，每次都要从宿主机中删除挂载的目录，这样能避免docker-entrypoint.sh中执行chown修改目录的所属用户

在进行下面的尝试之前，我们需要先修改config/log4j2.properties配置文件，让elasticsearch的日志可以写入到日志文件中。（Elasticsearch5.x版本使用了log4j2，默认是只将日志输出到控制台的，这里和Elasticsearch2.x版本不同）

```
status = error

appender.console.type = Console
appender.console.name = console
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] %marker%m%n

appender.rolling.type = RollingFile
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = [%d{ISO8601}][%-5p][%-25c] %.10000m%n
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}.log
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 1
appender.rolling.policies.time.modulate = true

rootLogger.level = info
rootLogger.appenderRef.console.ref = console
rootLogger.appenderRef.all.ref = rolling
```

### 尝试一：删掉Dockerfile和docker-entrypoint.sh的chown语句

```
# 构建修改后的Elasticsearch的Docker镜像
$ docker build -t "birdben/elasticsearch:5.3.1" .

# 运行Elasticsearch的Docker容器，并且挂载对应的data目录
$ docker run -itd -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/data:/usr/share/elasticsearch/data --name elasticsearch_me_5x birdben/elasticsearch:5.3.1

# 查看Docker容器的日志
$ docker logs 4353faea17cb
2017-05-04 06:07:39,353 main ERROR Unable to create file /usr/share/elasticsearch/logs/elasticsearch.log java.io.IOException: Permission denied
	at java.io.UnixFileSystem.createFileExclusively(Native Method)
	at java.io.File.createNewFile(File.java:1012)
	at org.apache.logging.log4j.core.appender.rolling.RollingFileManager$RollingFileManagerFactory.createManager(RollingFileManager.java:463)
	at org.apache.logging.log4j.core.appender.rolling.RollingFileManager$RollingFileManagerFactory.createManager(RollingFileManager.java:445)
	at org.apache.logging.log4j.core.appender.AbstractManager.getManager(AbstractManager.java:112)
	...

# 查看Docker容器内/usr/share/elasticsearch目录的权限
root@4353faea17cb:/usr/share/elasticsearch# ls -lh
total 228K
-rw-r--r--  1 root root 190K Apr 17 15:55 NOTICE.txt
-rw-r--r--  1 root root 9.4K Apr 17 15:55 README.textile
drwxr-xr-x  2 root root 4.0K May  4 06:05 bin
drwxr-xr-x  1 root root 4.0K May  4 06:06 config
drwxr-xr-x  3 root root  102 May  4 06:07 data
drwxr-xr-x  2 root root 4.0K May  4 06:05 lib
drwxr-xr-x  2 root root 4.0K May  4 06:05 logs
drwxr-xr-x 12 root root 4.0K May  4 06:05 modules
drwxr-xr-x  2 root root 4.0K Apr 17 15:55 plugins

# 查看Docker容器内/usr/share/elasticsearch/data目录的权限
$ ls -lh /usr/share/elasticsearch/data/
total 0
drwxr-xr-x 3 root root 102 May  4 06:33 nodes

# 查看Docker容器内/usr/share/elasticsearch/logs目录的权限（没有日志文件，因为上面没有权限的问题）
$ ls -lh /usr/share/elasticsearch/logs/
total 0

# 查看Elasticsearch进程
$ ps -ef | grep elasticsearch
elastic+     1     0 11 06:33 ?        00:00:19 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms2g -Xmx2g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/usr/share/elasticsearch -cp /usr/share/elasticsearch/lib/elasticsearch-5.3.1.jar:/usr/share/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
root       106    92  0 06:36 ?        00:00:00 grep elasticsearch
```

这里elasticsearch进程是属于elasticsearch用户的，而/usr/share/elasticsearch/data和/usr/share/elasticsearch/logs目录都属于root用户，所以没有权限在/usr/share/elasticsearch/logs目录下创建elasticsearch.log日志文件，这点理解起来比较容易。

```
# 新建文档
$ curl -XPOST 'http://127.0.0.1:9200/user/1/1' -d '{"name":"birdben"}'

# 查看索引文件
$ ls -lh /usr/share/elasticsearch/data/nodes/0/indices/ycOyM3onRbeZAPlYMVze_w/0/index/
total 4.0K
-rw-r--r-- 1 root root 130 May  4 06:38 segments_1
-rw-r--r-- 1 root root   0 May  4 06:38 write.lock
```

可以看出新建文档的索引文件所属用户也是root。

### 尝试二：只删掉docker-entrypoint.sh的chown语句

```
# 构建修改后的Elasticsearch的Docker镜像
$ docker build -t "birdben/elasticsearch:5.3.1" .

# 运行Elasticsearch的Docker容器，并且挂载对应的data目录
$ docker run -itd -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/data:/usr/share/elasticsearch/data --name elasticsearch_me_5x birdben/elasticsearch:5.3.1

# 查看Docker容器的日志，没有问题
$ docker logs 6db67f60ed6e

# 查看Docker容器内/usr/share/elasticsearch目录的权限
root@6db67f60ed6e:/usr/share/elasticsearch# ls -lh
total 228K
-rw-r--r--  1 root          root          190K Apr 17 15:55 NOTICE.txt
-rw-r--r--  1 root          root          9.4K Apr 17 15:55 README.textile
drwxr-xr-x  2 root          root          4.0K May  4 07:02 bin
drwxr-xr-x  1 elasticsearch elasticsearch 4.0K May  4 07:02 config
drwxr-xr-x  3 root          root           102 May  4 09:16 data
drwxr-xr-x  2 root          root          4.0K May  4 07:02 lib
drwxr-xr-x  1 elasticsearch elasticsearch 4.0K May  4 09:16 logs
drwxr-xr-x 12 root          root          4.0K May  4 07:02 modules
drwxr-xr-x  2 root          root          4.0K Apr 17 15:55 plugins

# 查看Docker容器内/usr/share/elasticsearch/data目录的权限
$ ls -lh /usr/share/elasticsearch/data/
total 0
drwxr-xr-x 3 root root 102 May  4 06:33 nodes

# 查看Docker容器内/usr/share/elasticsearch/logs目录的权限（生成日志文件了，但是logs目录的所属用户还是root，但是config和logs的所属用户却是elasticsearch，因为在Dockerfile中chown更改目录的所属用户后，又使用Volume挂载了data目录，而挂载的目录的所属用户就会被就修改为root用户，这也就解释了为什么data所属用户是root，config和logs所属用户是elasticsearch）
$ ls -lh /usr/share/elasticsearch/logs/
total 8.0K
-rw-r--r-- 1 elasticsearch elasticsearch 5.3K May  4 09:19 elasticsearch.log

# 查看Elasticsearch进程
$ ps -ef | grep elasticsearch
elastic+     1     0  4 09:16 ?        00:00:20 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms2g -Xmx2g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/usr/share/elasticsearch -cp /usr/share/elasticsearch/lib/elasticsearch-5.3.1.jar:/usr/share/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
root       104    94  0 09:23 ?        00:00:00 grep elasticsearch
```

这里elasticsearch进程也是属于elasticsearch用户的，而/usr/share/elasticsearch/data属于root用户，/usr/share/elasticsearch/config和/usr/share/elasticsearch/logs目录都属于elasticsearch用户，所以现在有权限在/usr/share/elasticsearch/logs目录下创建elasticsearch.log日志文件，这里猜测因为/usr/share/elasticsearch/logs没有挂载到宿主机，所以logs目录和目录下创建的elasticsearch.log日志文件都属于elasticsearch用户（因为Dockerfile中对logs目录进行了chown）。

```
# 新建文档
$ curl -XPOST 'http://127.0.0.1:9200/user/1/1' -d '{"name":"birdben"}'

# 查看索引文件
$ ls -lh /usr/share/elasticsearch/data/nodes/0/indices/meAhtSJXRl-cKzoqQZifBQ/0/index/
total 4.0K
-rw-r--r-- 1 root root 130 May  4 09:27 segments_1
-rw-r--r-- 1 root root   0 May  4 09:27 write.lock
```

可以看出新建文档的索引文件所属用户仍然是root。

下面我们证实一下我上面的猜测，/usr/share/elasticsearch/logs没有挂载到宿主机，所以logs目录和目录下创建的elasticsearch.log日志文件都属于elasticsearch用户，而不是root用户。这里推测一下，如果我把/usr/share/elasticsearch/logs挂载到宿主机，那logs目录和目录下的创建的elasticsearch.log日志文件就会属于root用户，而不是elasticsearch用户。（前提docker run的时候，-u使用的默认root用户，而不是elasticsearch用户）

### 尝试三：只删掉docker-entrypoint.sh的chown语句，然后挂载/usr/share/elasticsearch/logs目录

```
# 运行Elasticsearch的Docker容器，并且挂载对应的data目录
$ docker run -itd -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/data:/usr/share/elasticsearch/data -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/logs:/usr/share/elasticsearch/logs --name elasticsearch_me_5x birdben/elasticsearch:5.3.1

# 查看Docker容器的日志，没有问题
$ docker logs 17734a3549ad

# 查看Docker容器内/usr/share/elasticsearch目录的权限（果然和推测的一样，logs目录的所属用户变成了root）
root@17734a3549ad:/usr/share/elasticsearch# ls -lh
total 224K
-rw-r--r--  1 root          root          190K Apr 17 15:55 NOTICE.txt
-rw-r--r--  1 root          root          9.4K Apr 17 15:55 README.textile
drwxr-xr-x  2 root          root          4.0K May  4 07:02 bin
drwxr-xr-x  1 elasticsearch elasticsearch 4.0K May  4 07:02 config
drwxr-xr-x  3 root          root           102 May  4 09:37 data
drwxr-xr-x  2 root          root          4.0K May  4 07:02 lib
drwxr-xr-x  3 root          root           102 May  4 09:37 logs
drwxr-xr-x 12 root          root          4.0K May  4 07:02 modules
drwxr-xr-x  2 root          root          4.0K Apr 17 15:55 plugins

# 查看Docker容器内/usr/share/elasticsearch/data目录的权限（不变，和之前一样）
$ ls -lh /usr/share/elasticsearch/data/
total 0
drwxr-xr-x 3 root root 102 May  4 06:33 nodes

# 查看Docker容器内/usr/share/elasticsearch/logs目录的权限（这里也和推测的一样，logs目录下创建的elasticsearch.log日志文件也属于root用户）
$ ls -lh /usr/share/elasticsearch/logs/
total 8.0K
-rw-r--r-- 1 root root 4.3K May  4 09:39 elasticsearch.log

# 查看Elasticsearch进程（不变，和之前一样）
$ ps -ef | grep elasticsearch
elastic+     1     0  9 09:37 ?        00:00:18 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms2g -Xmx2g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/usr/share/elasticsearch -cp /usr/share/elasticsearch/lib/elasticsearch-5.3.1.jar:/usr/share/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
root       106    94  0 09:40 ?        00:00:00 grep elasticsearch
```

新建文档也和之前一样（忽略）

通过上面的尝试结果，可以得出如下结论：

- 即使在Dockerfile使用chown修改了目录的所属用户，但是只要目录被挂载到宿主机，则该目录的所属用户又会被修改为root用户。
- 如果不在Dockerfile中进行chown操作，当使用elasticsearch用户启动进程时，是无法访问root用户的目录的（目录被挂载后，目录的所属用户被修改为root用户的除外）

OK，前面的尝试隐藏了一点，我没有做特殊说明，就是我们前面的尝试都使用的root用户启动的容器。

```
# 这里我们没有指定-u或者--user参数，默认就是使用root用户启动容器
$ docker run -itd -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/data:/usr/share/elasticsearch/data --name elasticsearch_me_5x birdben/elasticsearch:5.3.1
```

所以docker-entrypoint.sh脚本中，有个if判断条件是不是root用户启动的容器"$(id -u)" = '0'，如果是root用户启动的容器，则使用gosu切换到elasticsearch启动elasticsearch进程。在这之前还进行了chown操作，将/usr/share/elasticsearch/data和/usr/share/elasticsearch/logs目录的所有者修改为elasticsearch用户。再回想下我们尝试一是把Dockerfile和docker-entrypoint.sh中的chown操作都删除掉了，所以elasticsearch进程才无法将日志写入到所属root用户的logs目录下。

```
#!/bin/bash

set -e

# Add elasticsearch as command if needed
if [ "${1:0:1}" = '-' ]; then
	set -- elasticsearch "$@"
fi

# Drop root privileges if we are running elasticsearch
# allow the container to be started with `--user`
if [ "$1" = 'elasticsearch' -a "$(id -u)" = '0' ]; then
	# Change the ownership of user-mutable directories to elasticsearch
	for path in \
		/usr/share/elasticsearch/data \
		/usr/share/elasticsearch/logs \
	; do
		chown -R elasticsearch:elasticsearch "$path"
	done
	
	set -- gosu elasticsearch "$@"
	#exec gosu elasticsearch "$BASH_SOURCE" "$@"
fi

# As argument is not related to elasticsearch,
# then assume that user wants to run his own process,
# for example a `bash` shell to explore this image
exec "$@"
```

这里可能有人会有疑问，那把Dockerfile中的chown操作删除，docker-entrypoint.sh中的chown操作加上的效果会不会和尝试二的结果一样呢？也可以正常将ES日志写入文件呢？我们再来尝试一下

### 尝试四：只删掉Dockerfile的chown语句，然后挂载/usr/share/elasticsearch/logs目录

```
# 构建修改后的Elasticsearch的Docker镜像
$ docker build -t "birdben/elasticsearch:5.3.1" .

# 运行Elasticsearch的Docker容器，并且挂载对应的data目录
$ docker run -itd -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/data:/usr/share/elasticsearch/data -v /Users/yunyu/workspace_git/birdDocker/elasticsearch/me/logs:/usr/share/elasticsearch/logs --name elasticsearch_me_5x birdben/elasticsearch:5.3.1

# 查看Docker容器的日志，没有问题
$ docker logs ca6eb8e80593

# 查看Docker容器内/usr/share/elasticsearch目录的权限（因为这里是在docker-entrypoint.sh对data和logs进行chown，所以只有data和logs所属elasticsearch用户）
root@ca6eb8e80593:/usr/share/elasticsearch# ls -lh
total 224K
-rw-r--r--  1 root          root          190K Apr 17 15:55 NOTICE.txt
-rw-r--r--  1 root          root          9.4K Apr 17 15:55 README.textile
drwxr-xr-x  2 root          root          4.0K May  4 10:54 bin
drwxr-xr-x  1 root          root          4.0K May  4 10:54 config
drwxr-xr-x  3 elasticsearch elasticsearch  102 May  4 10:58 data
drwxr-xr-x  2 root          root          4.0K May  4 10:54 lib
drwxr-xr-x  3 elasticsearch elasticsearch  102 May  4 10:58 logs
drwxr-xr-x 12 root          root          4.0K May  4 10:54 modules
drwxr-xr-x  2 root          root          4.0K Apr 17 15:55 plugins

# 查看Docker容器内/usr/share/elasticsearch/data目录的权限
$ ls -lh /usr/share/elasticsearch/data/
total 0
drwxr-xr-x 3 root root 102 May  4 10:58 nodes

# 查看Docker容器内/usr/share/elasticsearch/logs目录的权限
$ ls -lh /usr/share/elasticsearch/logs/
total 4.0K
-rw-r--r-- 1 root root 3.9K May  4 10:59 elasticsearch.log

# 查看Elasticsearch进程
$ ps -ef | grep elasticsearch
elastic+     1     0  4 10:58 ?        00:00:21 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms2g -Xmx2g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/usr/share/elasticsearch -cp /usr/share/elasticsearch/lib/elasticsearch-5.3.1.jar:/usr/share/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
root       107    96  0 11:06 ?        00:00:00 grep elasticsearch
```

```
# 新建文档
$ curl -XPOST 'http://127.0.0.1:9200/user/1/1' -d '{"name":"birdben"}'

# 查看索引文件
$ ls -lh /usr/share/elasticsearch/data/nodes/0/indices/meAhtSJXRl-cKzoqQZifBQ/0/index/
total 4.0K
-rw-r--r-- 1 root root 130 May  4 11:07 segments_1
-rw-r--r-- 1 root root   0 May  4 11:07 write.lock
```

这里data和logs目录都属于elasticsearch用户，但是data和logs目录下的文件却都属于root用户，这是什么情况呢？

因为docker run运行容器的时候，没有指定-u或者--user参数，这样就默认使用root用户启动容器，而在卷中创建的文件和文件夹将具有与在容器中创建它们的用户（root用户）相同的uid:gid（数字）。 如果你在容器内添加一个用户，具有与容器相同的uid:gid，并将其作为该用户（elasticsearch用户）运行，就可以使在卷中创建的文件和文件夹将具有与在容器中创建它们的用户（elasticsearch用户）相同的uid:gid（数字）。

所以这里docker-entrypoint.sh中的chown也很重要，因为只有root用户启动容器（docker run -u root）的时候会执行chown操作，如果是使用elasticsearch用户启动容器（docker run -u elasticsearch）的时候就不会执行chown操作。所以此种情况需要在Dockerfile中先执行chown操作。

总结如下：

volume挂载的目录默认属于root用户，如果没有chown给其他用户的话，在Volume卷中创建的文件和文件夹将具有与在容器中创建它们的用户相同的uid:gid（数字）。

参考文章：

- https://yq.aliyun.com/articles/53990
- https://github.com/moby/moby/issues/3124