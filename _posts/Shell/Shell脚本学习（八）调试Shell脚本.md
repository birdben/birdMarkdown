---
title: "Shell脚本学习（八）调试Shell脚本"
date: 2016-11-02 18:30:13
tags: [Shell]
categories: [Shell]
---

最近在在使用Jenkins做自动化部署的时候，仔细观察了一下Jenkins中执行Shell时会将每条Shell语句输出到控制台日志，这样调试起来Shell脚本非常方便

Jenkins的Shell执行方式

```
[birdben] $ /bin/sh -xe /tmp/hudson168932309618552744.sh
+ echo 'execute shell'
execute shell
....
```

实际上Jenkins执行Shell的方式只是多加了两个参数-xe

- -x : 跟踪调试Shell脚本
- -e : 表示一旦出错，就退出当前的Shell

"-x"选项可用来跟踪脚本的执行，是调试Shell脚本的强有力工具。"-x"选项使Shell在执行脚本的过程中把它实际执行的每一个命令行显示出来，并且在行首显示一个"+"号。"+"号后面显示的是经过了变量替换之后的命令行的内容，有助于分析实际执行的是什么命令。"-x"选项使用起来简单方便，可以轻松对付大多数的Shell调试任务,应把其当作首选的调试手段。

有的时候我们可能不希望输出全部的Shell命令，我们可以在Shell脚本中使用set设置需要跟踪的程序段，用下面的方式对需要调试的程序段进行跟踪，其他不在该程序段的命令不会被输出。

##### Shell脚本模板

```
set -x　　　 # 启动"-x"选项
要跟踪的程序段
set +x　　　 # 关闭"-x"选项
```

##### Shell脚本例子

```
#!bin/bash
docker ps -a

set -x
docker images
set +x

docker version
```

##### 执行Shell的结果

这里我们执行Shell脚本并没有带-x参数，但是可以看到docker ps -a和docker -version这两行Shell命令都没有输出，只有set语句中间的docker image命令输出了

```
$ sh aa.sh
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
+ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
birdben/jdk8        v1                  36fd8962f92c        10 months ago       656 MB
+ set +x
Client:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.7.1
 Git commit:   6f9534c
 Built:        Thu Sep  8 10:31:18 2016
 OS/Arch:      darwin/amd64

Server:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   23cf638
 Built:        Thu Aug 18 17:52:38 2016
 OS/Arch:      linux/amd64
```