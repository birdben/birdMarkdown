---
title: "Docker实战（二十六）Docker的run命令使用"
date: 2017-02-07 10:14:24
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

上一篇介绍了Dockerfile的配置详情用法，这里具体在讲解一下Docker的run命令使用，之前在构建自己的Docker镜像时，很多参数都没有深入研究，只是保证Docker容器的正常使用，这里run命令可以动态设置一些启动Docker容器的参数。


### Docker run命令格式

最基本的docker run命令的格式如下：

```
$ sudo docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```

之前第一篇Docker文章介绍过Docker的基础命令，其中也说明了run命令的一些参数，这里我们在回顾一下。

```
# 指定配置启动
$ sudo docker run -d -p 10.211.55.4:9999:22 birdben/ubuntu:v1 '/usr/sbin/sshd' -D

# 参数：
# -d：表示以“守护模式”执行，日志不会出现在输出终端上(输出结果可以用docker logs 查看)。使用 -d 参数启动后会返回一个唯一的 id，也可以通过 docker ps 命令来查看容器信息。
# -i：表示以“交互模式”运行容器，-i 则让容器的标准输入保持打开
# -t：表示容器启动后会进入其命令行，-t 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上
# -v：表示需要将本地哪个目录挂载到容器中，格式：-v <宿主机目录>:<容器目录>，-v 标记来创建一个数据卷并挂载到容器里。在一次 run 中多次使用可以挂载多个数据卷。
# -p：表示宿主机与容器的端口映射，此时将容器内部的 22 端口映射为宿主机的 9999 端口，这样就向外界暴露了 9999 端口，可通过 Docker 网桥来访问容器内部的 22 端口了。

# 注意：
# 这里使用的是宿主机的 IP 地址：10.211.55.4，与对外暴露的端口号 9999，它映射容器内部的端口号 22。ssh外部需要访问：ssh root@10.211.55.4 -p 9999
# 容器是否会长久运行，是和docker run指定的命令有关，和 -d 参数无关。
# run启动不一定要使用“镜像 ID”，也可以使用“仓库名:标签名”
```

注意：

这里在说明一下自己的之前构建的镜像的问题，这里不推荐在Docker中安装SSH，通过SSH来远程登录到Docker容器，应该通过在宿主机运行docker exec的方式进入Docker容器来进行操作，这样安全性更好，这是自己之前构建Docker镜像存在的一些问题。

### 容器识别

#### Name（--name）

可以通过三种方式为容器命名：

- 使用UUID长命名（"f78375b1c487e03c9438c729345e54db9d20cfa2ac1fc3494b6eb60872e74778"）
- 使用UUID短命令（"f78375b1c487"）
- 使用Name("evil_ptolemy")

这个UUID标示是由Docker deamon生成的。如果你在执行docker run时没有指定--name，那么deamon会自动生成一个随机字符串UUID。但是对于一个容器来说有个name会非常方便，当你需要连接其它容器时或者类似需要区分其它容器时，使用容器名称可以简化操作。无论容器运行在前台或者后台，这个名字都是有效的。

```
$ docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name docker_shadowsocks birdben/shadowsocks:v1
```

推荐运行Docker容器的时候指定一个容器名字，后续对于Docker容器的操作比较容易，否则每次使用Docker容器的ID操作都需要使用docker ps来查看比较麻烦。

### PID equivalent

如果在使用Docker时有自动化的需求，你可以将containerID输出到指定的文件中（PIDfile），类似于某些应用程序将自身ID输出到文件中，方便后续脚本操作。

```
$ docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --cidfile="/var/run/docker_shadowsocks.pid" --name docker_shadowsocks birdben/shadowsocks:v1
```

#### Image[:tag]

当一个镜像的名称不足以分辨这个镜像所代表的含义时，你可以通过tag将版本信息添加到run命令中，以执行特定版本的镜像。这个也是我之前比较常用的方式。

```
$ docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor birdben/shadowsocks:v1
```

#### Network Settings

默认情况下，所有的容器都开启了网络接口，同时可以接受任何外部的数据请求。这个是设置Docker网络配置的参数，如果你有多个Docker容器之前需要网络通信，那么这个参数是比较常用的。

```
--dns=[]         : Set custom dns servers for the container
--net="bridge"   : Set the Network mode for the container
  'bridge': creates a new network stack for the container on the docker bridge
  'none': no networking for this container
  'container:<name|id>': reuses another container network stack
  'host': use the host network stack inside the container
--add-host=""    : Add a line to /etc/hosts (host:IP)
--mac-address="" : Sets the container's Ethernet device's MAC address
```

你可以通过docker run --net none来关闭网络接口，此时将关闭所有网络数据的输入输出，你只能通过STDIN、STDOUT或者files来完成I/O操作。默认情况下，容器使用主机的DNS设置，你也可以通过--dns来覆盖容器内的DNS设置。同时Docker为容器默认生成一个MAC地址，你可以通过--mac-address 12:34:56:78:9a:bc来设置你自己的MAC地址。

Docker支持的网络模式有：

- none : 关闭容器内的网络连接
- bridge : 通过veth接口来连接容器，默认配置。
- host : 允许容器使用host的网络堆栈信息。 注意：这种方式将允许容器访问host中类似D-BUS之类的系统服务，所以认为是不安全的。
- container : 使用另外一个容器的网络堆栈信息。

#### 管理/etc/hosts

/etc/hosts文件中会包含容器的hostname信息，我们也可以使用--add-host这个参数来动态添加/etc/hosts中的数据。使用--add-host参数我们就可以动态配置hosts，而不需要在构建镜像的时候将/etc/hosts文件写好并且覆盖到Docker镜像中，这个也是我之前所犯下的错误。

```
$ /docker run -it --add-host mysql:192.168.2.108 birdben/shadowsocks:v1 cat /etc/hosts127.0.0.1	localhost192.168.2.109  hadoop1192.168.2.110  hadoop2192.168.2.111  hadoop3192.168.2.108  mysql
```

#### Clean up (--rm)

默认情况下，每个容器在退出时，它的文件系统也会保存下来，这样一方面调试会方便些，因为你可以通过查看日志等方式来确定最终状态。另外一方面，你也可以保存容器所产生的数据。但是当你仅仅需要短暂的运行一个容器，并且这些数据不需要保存，你可能就希望Docker能在容器结束时自动清理其所产生的数据。这个时候你就需要--rm这个参数了，--rm参数设置为true会在停止Docker容器的时候自动删除该容器的，在构建调试Docker容器的时候比较方便，省去手动反复删除Docker容器的操作，自己深有体会推荐构建调试的时候使用。

```
# 。
# 注意：--rm 和 -d不能共用！
$ docker run -itd --rm=true -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
Conflicting options: --rm and -d

# 正确用法，挂载到宿主机的日志文件是不会被删除的
$ docker run -it --rm=true -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
```

### 覆盖Dockerfile配置文件默认值

Docker run命令可以指定参数来覆盖Dockerfile中的配置，这些参数中，有四个是无法被覆盖的：FROM、MAINTAINER、RUN和ADD（这里可能是原作者写漏了，COPY也应该是无法被覆盖的），其余参数都可以通过docker run进行覆盖。我们将介绍如何对这些参数进行覆盖。

```
CMD (Default Command or Options)
ENTRYPOINT (Default Command to Execute at Runtime)
EXPOSE (Incoming Ports)
ENV (Environment Variables)
VOLUME (Shared Filesystems)
USER
WORKDIR
```

#### CMD

```
$ docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```

这个命令中的COMMAND部分是可选的。因为这个IMAGE在build时，开发人员可能已经设定了默认执行的命令。作为操作人员，你可以使用上面命令中新的command来覆盖旧的command。

如果镜像中设定了ENTRYPOINT，那么命令中的CMD也可以作为参数追加到ENTRYPOINT中。

```
# 这里使用/bin/bash命令覆盖原来启动supervisor的命令，这里是直接进入Docker容器了，并且检查supervisor进程并没有启动
$ docker run -it -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1 /bin/bash
root@77d46c117967:/# ps -ef | grep supervisor
root        16     1  0 03:54 ?        00:00:00 grep --color=auto supervisor
```

#### ENTRYPOINT

```
--entrypoint="": Overwrite the default entrypoint set by the image
```

这个ENTRYPOINT和COMMAND类似，它指定了当容器执行时，需要启动哪些进程。相对COMMAND而言，ENTRYPOINT是很难进行覆盖的，这个ENTRYPOINT可以让容器设定默认启动行为，所以当容器启动时，你可以执行任何一个二进制可执行程序。你也可以通过COMMAND为ENTRYPOINT传递参数。但当你需要在容器中执行其它进程时，你就可以指定其它ENTRYPOINT了。

下面就是一个例子，容器可以在启动时自动执行Shell，然后启动其它进程。

```
$ sudo docker run -i -t --entrypoint /bin/bash example/redis
#or two examples of how to pass more parameters to that ENTRYPOINT:
$ sudo docker run -i -t --entrypoint /bin/bash example/redis -c ls -l
$ sudo docker run -i -t --entrypoint /usr/bin/redis-cli example/redis --help
```

#### EXPOSE

```
--expose=[]: Expose a port or a range of ports from the container
        without publishing it to your host
-P=false   : Publish all exposed ports to the host interfaces
-p=[]      : Publish a container᾿s port to the host (format:
         ip:hostPort:containerPort | ip::containerPort |
         hostPort:containerPort | containerPort)
         (use 'docker port' to see the actual mapping)
--link=""  : Add link to another container (name:alias)
```

--expose可以让容器接受外部传入的数据。容器内监听的端口不需要和外部主机的端口相同。比如说在容器内部，一个HTTP服务监听在80端口，对应外部主机的端口就可能是49880.

如果使用-p或者-P，那么容器会开放部分端口到主机，只要对方可以连接到主机，就可以连接到容器内部。当使用-P时，Docker会在主机中随机从49153和65535之间查找一个未被占用的端口绑定到容器。你可以使用docker port来查找这个随机绑定端口。

当你使用--link方式时，作为客户端的容器可以通过私有网络形式访问到这个容器。同时Docker会在客户端的容器中设定一些环境变量来记录绑定的IP和PORT。

```
# 这里设置宿主机和Docker容器的端口都是443
$ docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
```

#### ENV

```
$ docker run -e MYVAR1 --env MYVAR2=foo --env-file ./env.list ubuntu bash
```

这将在容器中设置简单（非数组）环境变量。 为了说明这里显示所有三个标志。 其中-e，-env取一个环境变量和值，或者如果没有提供"="，那么通过export设置该变量的当前值（即，来自主机的$MYVAR1被设置为容器中的$MYVAR1） 。当没有提供"="且该变量没有在客户端环境中定义时，那个变量将从容器的环境变量列表中删除。所有三个标志，-e，--env和--env文件可以重复。

不管这三个标志的顺序如何，首先处理--env文件，然后处理-e，-env标志。 这样，-e或--env将根据需要覆盖变量。

```
$ cat ./env.list
TEST_FOO=BAR
$ docker run --env TEST_FOO="This is a test" --env-file ./env.list busybox env | grep TEST_FOO
TEST_FOO=This is a test
```

--env-file标志采用文件名作为参数，并且期望每一行都处于VAR = VAL格式，模拟传递给--env的参数。 注释行只需要前缀为＃

使用--env-file传递的文件示例

```
$ cat ./env.list
TEST_FOO=BAR

# this is a comment
TEST_APP_DEST_HOST=10.10.0.127
TEST_APP_DEST_PORT=8888
_TEST_BAR=FOO
TEST_APP_42=magic
helloWorld=true
123qwe=bar
org.spring.config=something

# pass through this variable from the caller
TEST_PASSTHROUGH
$ TEST_PASSTHROUGH=howdy docker run --env-file ./env.list busybox env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=5198e0745561
TEST_FOO=BAR
TEST_APP_DEST_HOST=10.10.0.127
TEST_APP_DEST_PORT=8888
_TEST_BAR=FOO
TEST_APP_42=magic
helloWorld=true
TEST_PASSTHROUGH=howdy
HOME=/root
123qwe=bar
org.spring.config=something

$ docker run --env-file ./env.list busybox env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=5198e0745561
TEST_FOO=BAR
TEST_APP_DEST_HOST=10.10.0.127
TEST_APP_DEST_PORT=8888
_TEST_BAR=FOO
TEST_APP_42=magic
helloWorld=true
TEST_PASSTHROUGH=
HOME=/root
123qwe=bar
org.spring.config=something
```

#### VOLUME

通过"-v"参数来覆盖挂载路径，这个是比较常用的，因为大多数日志或者数据文件都会挂载到宿主机目录的。

```
-v=[]: Create a bind mount with: [host-dir]:[container-dir]:[rw|ro].
   If "container-dir" is missing, then docker creates a new volume.
--volumes-from="": Mount all volumes from the given container(s)
```

```
# 宿主机目录：/Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks
# Docker容器目录：/var/log/shadowsocks
$ docker run -itd -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
```

#### USER

容器中默认的用户是root，但是开发人员创建新的用户之后，这些新用户也是可以使用的。开发人员可以通过Dockerfile的USER设定默认的用户，并通过"-u"来覆盖这些参数。

```
# 这里我通过-u参数指定的shadowsocks用户来运行，但是我并没有创建过shadowsocks用户，所以下面报错了
$ docker run -it --rm=true -u="shadowsocks" -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
docker: Error response from daemon: linux spec user: unable to find user shadowsocks: no matching entries in passwd file.
```

#### WORKDIR

容器中默认的工作目录是根目录（/）。开发人员可以通过Dockerfile的WORKDIR来设定默认工作目录，操作人员可以通过"-w"来覆盖默认的工作目录。

```
# -w指定默认工作目录是"/usr/local"
$ docker run -itd -w="/usr/local" -p 443:443 -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/shadowsocks:/var/log/shadowsocks -v /Users/yunyu/workspace_git/birdDocker/shadowsocks/logs/supervisor:/var/log/supervisor --name shadowsocks_docker birdben/shadowsocks:v1
358618f9aa4d1c388b3dc9c7e5c6236bb9b6b150728e474710e83f0409dd6007

$ docker exec -it 358618f9aa4d /bin/bash
root@358618f9aa4d:/usr/local#
```

这里我只记录了自己使用Docker常用的配置，更多详细配置用法请看参考文章，谢谢!

参考文章：

- http://dockone.io/article/152
- https://docs.docker.com/engine/reference/commandline/run/