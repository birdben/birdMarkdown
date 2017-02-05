
---
title: "Docker实战（二十三）Docker安装Shadowsocks"
date: 2017-01-19 12:01:57
tags: [Docker环境]
categories: [Docker]
---

最近DO服务器一直没续费到期了，想换个新的VPS服务器又发现原来的翻墙工具Shadowsocks又要重装好麻烦啊。所以想到了用Docker安装一个Shadowsocks镜像的一劳永逸的方式。

Dockerfile文件

```
############################################
# version : birdben/shadowsocks:v1
# desc : 当前版本安装的shadowsocks
############################################
# 设置继承自我们创建的 tools 镜像
FROM birdben/tools:v1

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 设置环境变量，所有操作都是非交互式的
ENV DEBIAN_FRONTEND noninteractive
ENV SSS_USER=shadowsocks
ENV SSS_SUPERVISOR_LOG_DIR=/var/log/supervisor
ENV SSS_SHADOWSOCKS_LOG_DIR=/var/log/shadowsocks

# Add a user and make dirs
RUN set -x \
    && useradd $SSS_USER \
    && mkdir -p $SSS_SUPERVISOR_LOG_DIR \
    && mkdir -p $SSS_SHADOWSOCKS_LOG_DIR \
    && chown $SSS_USER:$SSS_USER $SSS_SUPERVISOR_LOG_DIR \
    && chown $SSS_USER:$SSS_USER $SSS_SHADOWSOCKS_LOG_DIR \
    && chmod 777 /var/run

# 替换 sources.list 的配置文件，并复制配置文件到对应目录下面。
# 这里使用的AWS国内的源，也可以替换成其他的源（例如：阿里云的源）
COPY sources.list /etc/apt/sources.list

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY shadowsocks.json /etc/shadowsocks.json

RUN sudo rm -rf /var/lib/apt/lists/*
RUN sudo apt-get update

RUN sudo apt-get -y install python-pip
RUN sudo pip install --upgrade pip
RUN sudo pip install shadowsocks

# /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor
# /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks
# 这里挂载的路径是birdTracker项目的目录
VOLUME ["/var/log/supervisor"]
VOLUME ["/var/log/shadowsocks"]

USER root

# 容器需要开放Shadowsocks的443端口
EXPOSE 443

# 执行run.sh文件
# CMD ["/run.sh"]
# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

Dockerfile源文件链接：

- https://github.com/birdben/birdDocker/blob/master/shadowsocks/Dockerfile

supervisord.conf

```
[program:shadowsocks]
command=ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
stderr_logfile = /var/log/shadowsocks/shadowsocks.err
stdout_logfile = /var/log/shadowsocks/shadowsocks.out
user=root
stopsignal=INT

[supervisord]
nodaemon=true
user=root
```

shadowsocks.json配置文件

```
{
    "server":"0.0.0.0",
    "server_port":443,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"123456",
    "timeout":500,
    "method":"aes-256-cfb",
    "fast_open":false
}

# 配置内容描述
server : 服务端监听的地址，服务端可填写 0.0.0.0
server_port : 服务端的端口
local_address : 本地端监听的地址
local_port : 本地端的端口
password : 用于加密的密码
timeout : 超时时间，单位秒
method : 默认"aes-256-cfb"，建议chacha20或者rc4-md5，因为这两个速度快
fast_open : 是否使用 TCP_FASTOPEN, true / false（后面优化部分会打开系统的 TCP_FASTOPEN，所以这里填 true，否则填 false)
```

构建Docker镜像

```
docker build -t "birdben/shadowsocks:v1" .
```

运行Docker容器

```
CURRENT_UID=`whoami`
docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_${CURRENT_UID} birdben/shadowsocks:v1
```

Shadowsocks客户端配置

```
地址：192.168.99.127
端口：443
加密：aes-256-cfb
密码：123456
```

也可以不使用配置文件方式启动，可以在启动的时候指定配置信息

```
ssserver -s 监听地址 -p 服务器端口 -k 密码 -m 加密方法

参数说明：
-s : 监听的服务器端地址
-p : 服务器监听端口
-k : 客户端连接密码
-m : 加密方式
--user : 指定启动进程的用户
--d : 指定运行方式，启动/关闭/重启
```

浏览器访问www.google.com就可以看到Shadowsocks的日志中会有访问记录出现

```
INFO: loading config from /etc/shadowsocks.json
2017-02-05 08:19:32 INFO     loading libcrypto from libcrypto.so.1.0.0
2017-02-05 08:19:32 INFO     starting server at 0.0.0.0:443
2017-02-05 08:21:46 INFO     connecting www.google.com:443 from 172.17.0.1:54762
2017-02-05 08:21:46 INFO     connecting www.facebook.com:443 from 172.17.0.1:54766
```


参考文章：

- https://blog.phpgao.com/shadowsocks_on_linux.html
- https://blog.phpgao.com/supervisor_shadowsocks.html
