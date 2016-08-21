---
title: "Docker实战（二）制作自己的Docker镜像"
date: 2015-11-17 00:54:35
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

制作自己的Docker镜像主要有如下两种方式：

### 1.使用docker commit 命令来创建镜像

1. 通过docker run命令启动容器
2. 修改docker镜像内容
3. docker commit提交修改的镜像
4. docker run新的镜像

### 2.使用 Dockerfile 来创建镜像

使用 docker commit 来扩展一个镜像比较简单，但是不方便在一个团队中分享。我们可以使用 docker build 来创建一个新的镜像。为此，首先需要创建一个 Dockerfile，包含一些如何创建镜像的指令。

#### Dockerfile 基本的语法

- 使用#来注释
- FROM 指令告诉 Docker 使用哪个镜像作为基础
- 接着是维护者的信息
- RUN开头的指令会在创建中运行，比如安装一个软件包，在这里使用 apt-get 来安装了一些软件

#### 构建镜像的步骤

1.新建一个目录和一个 Dockerfile

```
$ mkdir new_folder
$ cd new_folder
$ touch Dockerfile
```

2.编写Dockerfile，Dockerfile中每一条指令都创建镜像的一层，例如：

```
# 这里是注释
# 设置继承自哪个镜像
FROM ubuntu:14.04
# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)
# 在终端需要执行的命令
RUN apt-get install -y openssh-serverRUN mkdir -p /var/run/sshd
```

3.编写完成 Dockerfile 后可以使用 docker build 来生成镜像。

```
$ sudo docker build -t="birdben/ubuntu:v1" .
# 下面是一堆构建日志信息

############
我是日志
############

# 参数：
# -t 标记来添加 tag，指定新的镜像的用户和镜像名称信息。 
# “.” 是 Dockerfile 所在的路径（当前目录），也可以替换为一个具体的 Dockerfile 的路径。

# 以交互方式运行docker
$ docker run -it birdben/ubuntu:v1 /bin/bash

# 运行docker时指定配置
$ sudo docker run -d -p 10.211.55.4:9999:22 ubuntu:tools '/usr/sbin/sshd' -D

# 参数：
# -i：表示以“交互模式”运行容器，-i 则让容器的标准输入保持打开
# -t：表示容器启动后会进入其命令行，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上
# -v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>，-v 标记来创建一个数据卷并挂载到容器里。在一次 run 中多次使用可以挂载多个数据卷。
# -p：指定对外80端口
# 不一定要使用“镜像 ID”，也可以使用“仓库名:标签名”

```

### Docker网络

Docker的网络功能相对简单，没有过多复杂的配置，Docker默认使用birdge桥接方式与容器通信，启动Docker后，宿主机上会产生docker0这样一个虚拟网络接口， docker0不是一个普通的网络接口， 它是一个虚拟的以太网桥，可以为绑定到docker0上面的网络接口自动转发数据包，这样可以使容器与宿主机之间相互通信。每次Docker创建一个容器，会产生一对虚拟接口，在宿主机上执行ifconfig，会发现多了一个类似veth****这样的网络接口，它会绑定到docker0上，由于所有容器都绑定到docker0上，容器之间也就可以通信。

在宿主机上执行ifconfig，会看到docker0这个网络接口， 启动一个container，再次执行ifconfig, 会有一个类似veth****的interface，每个container的缺省路由是宿主机上docker0的ip，在container中执行netstat -r可以看到如下图所示内容：
container路由

> 在容器中使用netstat -r命令查看容器的IP地址

![](http://tech.meituan.com/img/docker_introduction/container_route.png)

容器中的默认网关跟docker0的地址是一样的：

> 在宿主机中使用ifconfig查看docker0的IP地址

![](http://tech.meituan.com/img/docker_introduction/host_docker0.png)

docker0
当容器退出之后，veth*虚拟接口也会被销毁。


#### 参考文章：

- http://blog.saymagic.cn/2015/06/01/learning-docker.html