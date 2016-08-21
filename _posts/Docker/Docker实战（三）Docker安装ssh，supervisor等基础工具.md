---
title: "Docker实战（三）Docker安装ssh，supervisor等基础工具"
date: 2015-11-17 00:55:43
tags: [Docker环境, Dockerfile]
categories: [Docker]
---

需要提前下载好官方的ubuntu镜像，我这里使用的是ubuntu:14.04版本，这里安装了一些基础的工具ssh，curl，wget，vim等等，包括后续的Docker镜像需要启动多个服务，所以提前先装好supervisor。

##### Dockerfile文件
```
############################################
# version : birdben/tools:v1
# desc : 当前版本安装的ssh，wget，curl，supervisor 
############################################
# 设置继承自ubuntu官方镜像
FROM ubuntu:14.04

# 下面是一些创建者的基本信息
MAINTAINER birdben (191654006@163.com)

# 注意这里要更改系统的时区设置，因为在 web 应用中经常会用到时区这个系统变量，默认的 ubuntu 会让你的应用程序发生不可思议的效果哦
ENV DEBIAN_FRONTEND noninteractive

# 清空ubuntu更新包
RUN sudo rm -rf /var/lib/apt/lists/*

# 一次性安装vim，wget，curl，ssh server等必备软件
# RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe"> /etc/apt/sources.list
RUN sudo apt-get update
RUN sudo apt-get install -y vim wget curl openssh-server sudo
RUN sudo mkdir -p /var/run/sshd

# 将sshd的UsePAM参数设置成no
RUN sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config

# 添加测试用户admin，密码admin，并且将此用户添加到sudoers里
RUN useradd admin
RUN echo "admin:admin" | chpasswd
RUN echo "admin   ALL=(ALL)       ALL" >> /etc/sudoers

# 把admin用户的shell改成bash，否则SSH登录Ubuntu服务器，命令行不显示用户名和目录 
RUN usermod -s /bin/bash admin

# 安装supervisor工具
RUN sudo apt-get install -y supervisor
RUN sudo mkdir -p /var/log/supervisor

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# 容器需要开放SSH 22端口
EXPOSE 22

# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]```

##### Dockerfile源文件链接：

- https://github.com/birdben/birdDocker/blob/master/tools/Dockerfile

##### supervisor配置文件内容

```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 2 个服务。每一段包含一个服务的目录和启动这个服务的命令。

[supervisord]
nodaemon=true

[program:sshd]
command=/usr/sbin/sshd -D
```

##### supervisor源文件链接：

- https://github.com/birdben/birdDocker/blob/master/tools/supervisord.conf

##### 控制台终端
```
# 构建镜像
$ docker build -t="birdben/tools:v1" .
# 运行已经构件好的镜像，因为我使用的ubuntu虚拟机安装的Docker，而我的虚拟机也安装了ssh服务，所以这里指定了宿主机的端口为9999对应Docker容器的22端口
$ docker run -p 9999:22 -t -i "birdben/tools:v1"


# 此时查看宿主机的9999端口，已经处于监听状态：
$ netstat -aunpt(Not all processes could be identified, non-owned process info will not be shown, you would have to be root to see it all.)Active Internet connections (servers and established)Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program nametcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      -               tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -               tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      -               tcp6       0      0 :::9999                 :::*                    LISTEN      -               tcp6       0      0 :::22                   :::*                    LISTEN      -   

# 再查看一下宿主机的IP地址，我这里的IP地址是10.211.55.4
$ ifconfig

# 此时可以通过ssh远程连接Docker容器了
$ ssh root@10.211.55.4 -p 9999
# 输入密码应该就可以连接到Docker容器了

# 如果遇到下面的问题，这是Linux重装或则openssh-server重装引起的，执行以下命令即可
$ ssh-keygen -R 10.211.55.4

# 如果上述方式不好用，进入此目录，删除的10.211.55.4相关rsa的信息即可
$ vi ~/.ssh/known_hosts


@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
8c:4b:88:88:53:4a:b1:f0:e2:da:9a:dc:aa:67:46:df.
Please contact your system administrator.
Add correct host key in /Users/ben/.ssh/known_hosts to get rid of this message.
Offending RSA key in /Users/ben/.ssh/known_hosts:18
RSA host key for [10.211.55.4]:9999 has changed and you have requested strict checking.
Host key verification failed.
```

##### 遇到的问题和解决办法
```
Q:ssh登录后，命令行不显示用户名和目录
A:把用户的shell改成bash，否则SSH登录Ubuntu服务器，命令行不显示用户名和目录
RUN usermod -s /bin/bash admin

参考：
http://bbs.csdn.net/topics/390188284

Q:ssh创建admin登录用户，不使用root登录
A:这里使用ssh不建议直接使用root用户登录，建议创建一个新的用户例如admin登录
RUN useradd admin
RUN echo "admin:admin" | chpasswd
RUN echo "admin   ALL=(ALL)       ALL" >> /etc/sudoers

参考：
http://blog.csdn.net/kongxx/article/details/38395305
http://blog.csdn.net/kongxx/article/details/38412119

Q:如何修改ssh服务相关配置
A:可以直接修改sshd_config配置文件
vi /etc/ssh/sshd_config
需要修改如下

# 设置不允许root用户登录
PermitRootLogin yes

# 利用 PAM 管理使用者认证有很多好处，可以记录与管理。
# 所以这里我们建议你使用 UsePAM 且 ChallengeResponseAuthentication 设定为 no，但是我们这里为了简单设置为密码认证，ChallengeResponseAuthentication设定为yes，UsePAM设置为no
ChallengeResponseAuthentication yes
UsePAM no

参考：
http://my.oschina.net/fsmwhx/blog/143354
http://www.cnblogs.com/ggjucheng/archive/2012/08/19/2646032.html
```


参考文章：

- http://blog.csdn.net/delphiwcdj/article/details/42836639
- http://www.cnblogs.com/linxiong945/p/4180565.html
- http://dockerpool.com/article/1414384697
- http://blog.csdn.net/kongxx/article/details/42528423
- http://udn.yyuap.com/doc/docker_practice/cases/supervisor.html

