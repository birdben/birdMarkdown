---
title: "Shell脚本学习（七）Shell中的特殊用法"
date: 2016-10-22 17:53:17
tags: [Shell]
categories: [Shell]
---

最近在网上看了别人写的Shell脚本，发现还是有很多语法看不懂需要百度才行，今天就总结一下我遇到的一些Shell特殊符号的用法问题

### Shell的特殊符号 $, $$, &, && 的用法


```
$$ : Shell本身的PID（ProcessID）
$! : Shell最后运行的后台Process的PID
$? : 最后运行的命令的结束代码（返回值）
$- : 使用Set命令设定的Flag一览，显示Shell使用的当前选项。
$* : Shell的所有参数列表
$@ : Shell的所有参数列表
$# : Shell的所有参数个数
$0 : Shell本身的文件名
$1～$n : Shell的各个参数值。$1是第1参数、$2是第2参数…
` : 反引号。反引号括起来的字符串被shell解释为命令行，在执行时，shell首先执行该命令行，并以它的标准输出结果取代整个反引号（包括两个反引号）部分。即`command`和$(command)的含义相同，都返回当前执行命令的结果。
& : 放在启动参数后面表示设置此进程为后台进程
| : 管道 (pipeline) 连结上个指令的标准输出，做为下个指令的标准输入
&& : Shell命令之间使用 && 连接，实现逻辑与的功能
|| : Shell命令之间使用 || 连接，实现逻辑或的功能
1.命令之间使用 && 连接，实现逻辑与的功能。
2.如果左边的命令有返回值，该返回值保存在Shell变量 $? 中，只有在 && 左边的命令返回真（命令返回值 $? == 0），&& 右边的命令才会被执行。
3.只要有一个命令返回假（命令返回值 $? == 1），表示左边的命令执行失败，后面的命令就不会被执行。
下一条命令依赖前一条命令是否执行成功。如：在成功地执行一条命令之后再执行另一条命令，或者在一条命令执行失败后再执行另一条命令等。shell 提供了 && 和 || 来实现命令执行控制的功能，shell 将根据 && 或 || 前面命令的返回值来控制其后面命令的执行。
```

举例说明上述的Shell特殊符号的用法

```
#!/bin/bash

#
# AUTHOR: Yanpeng Lin
# DATE:   Mar 30 2014
# DESC:   lock a rotating file(static filename) and tail
#

PID=$( mktemp )
echo $PID
echo $(eval "cat $PID")
while true;
do
    CURRENT_TARGET=$( eval "echo $1" )
    echo $CURRENT_TARGET
    if [ -e ${CURRENT_TARGET} ]; then
        IO=`stat ${CURRENT_TARGET}`
        # 在后台运行监听{$CURRENT_TARGET}文件的变化，如果出错不输出错误信息，将最后执行的后台进程的ID输出到${PID}中（也就是tail -f {$CURRENT_TARGET} 2> /dev/null &这个命令的后台进程ID）
        tail -f {$CURRENT_TARGET} 2> /dev/null & echo $! > $PID;
        echo $!
    fi
	echo $PID
	echo $(eval "cat $PID")
    
    # as long as the file exists and the inode number did not change
    while [[ -e ${CURRENT_TARGET} ]] && [[ ${IO} = `stat -c %i ${CURRENT_TARGET}` ]]
    do
        CURRENT_TARGET=$( eval "echo $1" )
        #echo $CURRENT_TARGET
        sleep 0.5
    done
    # 如果kill命令执行失败，则输出错误信息，并且不会清空${PID}中的值
    if [ ! -z ${PID} ]; then
        kill `cat ${PID}` 2> /dev/null && echo > ${PID}
    fi
    sleep 0.5
done 2> /dev/null
rm -rf ${PID}
```


```
${var:-default} : 使用一个默认值（一般是空值）来代替那些空的或者没有赋值的变量var
${var:=default} : 使用指定值来代替空的或者没有赋值的变量var
${var:?message} : 如果变量为空或者未赋值，那么会显示出错误信息并终止脚本的执行同时返回退出码1
${#var} : 给出var的长度
${var%pattern} : 表示从var最右边(即结尾)开始删除与pattern匹配的最小部分，然后返回剩余部分
${var%%pattern} : 表示从var最右边(即结尾)开始删除与pattern匹配的最长部分，然后返回剩余部分
${var#pattern} : 表示从var最左边(即开始)开始删除与pattern匹配的最小部分，然后返回剩余部分
${var##pattern} : 表示从var最左边(即开始)开始删除与pattern匹配的最长部分，然后返回剩余部分

注意：只有在pattern中使用了通配符才能有最长最短的匹配，否则没有最长最短匹配之分。

例如：
${1#-} : ${1#-}是判断第一个参数是否以"-"开头
${1%.conf} : ${1%.conf}是判断第一个参数是否以".conf"结尾
```

test.sh脚本

```
echo ${test_path:-}
echo ${test_path:=test}
# 运行会报错error_message
# echo ${test_path1:?error_message}
echo ${#test_path}

test_pattern="stxxa_styya_stzzd"
echo ${test_pattern#st*a}
echo ${test_pattern##st*a}

test_pattern="ccc_aaab.conf_ab.conf"
echo ${test_pattern%a*.conf}
echo ${test_pattern%%a*.conf}
```

运行结果

```
（空）
test
4
_styya_stzzd
_stzzd
ccc_aaab.conf_
ccc_
```


再看一个DockerHub上的Redis官方的Dockerfile和Shell脚本

Dockerfile文件

```
FROM debian:jessie

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r redis && useradd -r -g redis redis

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        wget \
    && rm -rf /var/lib/apt/lists/*

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.7
RUN set -x \
    && wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
    && wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
    && export GNUPGHOME="$(mktemp -d)" \
    && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
    && gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
    && rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
    && chmod +x /usr/local/bin/gosu \
    && gosu nobody true

ENV REDIS_VERSION 3.2.8
ENV REDIS_DOWNLOAD_URL http://download.redis.io/releases/redis-3.2.8.tar.gz
ENV REDIS_DOWNLOAD_SHA1 6780d1abb66f33a97aad0edbe020403d0a15b67f

# for redis-sentinel see: http://redis.io/topics/sentinel
RUN set -ex \
    \
    && buildDeps=' \
        gcc \
        libc6-dev \
        make \
    ' \
    && apt-get update \
    && apt-get install -y $buildDeps --no-install-recommends \
    && rm -rf /var/lib/apt/lists/* \
    \
    && wget -O redis.tar.gz "$REDIS_DOWNLOAD_URL" \
    && echo "$REDIS_DOWNLOAD_SHA1 *redis.tar.gz" | sha1sum -c - \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && rm redis.tar.gz \
    && grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 1$' /usr/src/redis/src/server.h \
    && sed -ri 's!^(#define CONFIG_DEFAULT_PROTECTED_MODE) 1$!\1 0!' /usr/src/redis/src/server.h \
    && grep -q '^#define CONFIG_DEFAULT_PROTECTED_MODE 0$' /usr/src/redis/src/server.h \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    \
    && rm -r /usr/src/redis \
    \
    && apt-get purge -y --auto-remove $buildDeps

RUN mkdir /data && chown redis:redis /data
VOLUME /data
WORKDIR /data

COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD [ "redis-server" ]
```

docker-entrypoint.sh脚本文件

```
#!/bin/sh
set -e

# first arg is `-f` or `--some-option`
# or first arg is `something.conf`
if [ "${1#-}" != "$1" ] || [ "${1%.conf}" != "$1" ]; then
    set -- redis-server "$@"
fi

# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    chown -R redis .
    exec gosu redis "$0" "$@"
fi

exec "$@"
```

这里Redis的Docker容器使用ENTRYPOINT的方式启动，ENTRYPOINT配置容器启动后执行的命令，构建镜像build时不执行，并且不可被 docker run 提供的参数覆盖。这里docker run的参数"redis-server"会添加到ENTRYPOINT后面，就成了这样docker-entrypoint.sh "redis-server"。相当于下面的两条命令的效果是一样，因为默认的CMD参数是"redis-server"。

```
# redis:3.2是Redis的Docker镜像名称

$ docker run -p 6379:6379 -d redis:3.2
$ docker run -p 6379:6379 -d redis:3.2 redis-server
```

当然我们也可以指定额外的启动参数，如下：

```
# 这里我们启动的时候指定了挂载的配置文件
$ docker run -v /Users/yunyu/Downloads/redis:/data -p 6379:6379 -d redis:3.2 redis-server /data/conf/redis.conf
```

具体在看一下docker-entrypoint.sh脚本的实现

```
# 这里是${1#-}是判断第一个参数是否以"-"开头，${1%.conf}是判断第一个参数是否以".conf"结尾
if [ "${1#-}" != "$1" ] || [ "${1%.conf}" != "$1" ]; then
    # "set -- redis-server"的意思是把"redis-server"加入到原来的参数列表中，并且放在参数列表中的第一个，可以通过"$1"获取，其他参数获取索引顺延
    # 这里把"redis-server"加入到原来的参数列表并且放在一个位置，是因为如果docker run传递了参数，而且第一个参数是以"-"开始，就说明没有redis-server参数，需要添加到参数列表的第一参数位置。
    set -- redis-server "$@"
fi

...

# 这里是把所有参数列表组成命令给执行器，执行该命令
# 也就是如果参数中没有redis-server，表示用户希望运行自己的其他进程
exec "$@"
```

通过下面的小例子来体会"set"和"set --"区别，"set --"可以将后面参数的"-"转义，不当成命令选项来解析，当成一个普通参数来解析。

```
$ set -- -z 2 3 4
$ echo $1
-z

$ set -z 2 3 4
set: bad option: -z
```



源文件连接地址：

- https://github.com/docker-library/redis/blob/3f926a47370a19fc88d57d0245823758cbf19b2d/3.2/Dockerfile
- https://github.com/docker-library/redis/blob/3f926a47370a19fc88d57d0245823758cbf19b2d/3.2/docker-entrypoint.sh

参考文章：

- http://unix.stackexchange.com/questions/308260/what-does-set-do-in-this-dockerfile-entrypoint
- http://www.360doc.com/content/13/0524/12/7044580_287728900.shtml
