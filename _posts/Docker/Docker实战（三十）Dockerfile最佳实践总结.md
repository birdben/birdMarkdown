---
title: "Docker实战（三十）Dockerfile最佳实践总结"
date: 2017-05-07 14:52:44
tags: [Dockerfile]
categories: [Docker]
---

这次重构Docker镜像也参考了网上许多关于Dockerfile编写的建议和技巧。本文主要翻译了官方给出的Dockerfile编写的建议，以及总结了一些网上Dockerfile编写的建议和技巧。

### 一般准则和建议

#### 容器应该是"短暂的"

由Dockerfile定义的image生成的容器应尽可能短暂。 通过"短暂的"，我们意味着容器它可以被stop和destroyed，一个新的容器的构建可以使用绝对最小的设置和配置。

#### 使用.dockerignore

在大多数情况下，最好将每个Dockerfile放在一个空目录中。 然后，仅添加构建Dockerfile所需的文件。 要增加构建的性能，可以通过将.dockerignore文件添加到该目录来排除文件和目录。

#### 避免安装不必要的安装包

应该尽量减少容器的复杂性，依赖性，文件的大小，构建的次数，所以应该尽量避免安装不必要的安装包。

#### 每个容器应该只有一个进程 "one process per container"

将应用程序解耦到多个容器中可以更轻松地水平扩展和重新使用容器。 例如，Web应用程序堆栈可能由三个独立的容器组成，每个容器具有自己独特的映像，以解耦的方式管理web application, database, memory cache。

如果容器之间有依赖关系，应该使用Docker Network解决容器之间的通信。

#### 最小化镜像的层数

在Dockerfile可读性和保持最少数据层之间找到平衡。一定要慎重引入新的数据层。

#### 排序多行参数

只要有可能，通过以安装的软件包的字母数字来排序。 这将帮助你避免重复的包，并使列表更容易更新。 这也使得PR更容易阅读和审查。 在反斜杠（\）之前添加空格也有帮助。

#### 构建缓存

在构建image的过程中，Docker将按照指定的顺序逐步执行你的Dockerfile中的指令。随着每条指令的检查，Docker将在其缓存中查找可重用的现有image，而不是创建一个新的（重复）image。如果你不想使用缓存，可以在docker build命令中使用--no-cache=true选项。

但是，如果你确实让Docker使用其缓存，那么了解何时会找到匹配的image是非常重要的。 Docker将遵循的基本规则如下：

- 从基础image开始就已经在缓存中了，将下一条指令与从该基础image导出的所有子image进行比较，以查看其中一条是否使用完全相同的指令构建。如果没有，则缓存无效。

- 在大多数情况下，只需将Dockerfile中的指令与其中一个子image进行比较即可。但是，某些说明需要更多的检查和解释。

- 对于ADD和COPY指令，将检查image中文件的内容，并为每个文件计算校验和。在这些校验和中不考虑文件的最后修改和最后访问的时间。在缓存查找期间，将校验和与现有image中的校验和进行比较。如果文件（如内容和元数据）中有任何变化，则缓存无效。

- 除了ADD和COPY命令之外，缓存检查将不会查看容器中的文件来确定缓存匹配。例如，当处理RUN apt-get -y update命令时，不会检查在容器中更新的文件以确定是否存在高速缓存命中。在这种情况下，只需使用命令字符串本身来查找匹配。

一旦缓存无效，所有后续的Dockerfile命令将生成新的映像，并且高速缓存将不被使用。

### Dockerfile的一些建议

#### FROM

只要有可能，使用当前的官方存储库作为你的image的基础。 我们建议使用Debian镜像，因为它是非常严格的控制，并保持最小（目前在150 mb），而仍然是一个完整的分布。

#### LABEL

给image添加label标签，能够更好的按照项目组织image信息，这个暂时没用到过。

#### RUN

可以在多行上分隔长度或复杂的RUN语句，并以反斜杠分隔。

应该避免运行RUN apt-get upgrade or dist-upgrade，因为基本映像中的许多"essential"程序包将无法在非特权容器内升级。尽量使用apt-get install -y foo更新一个特定的包。

请务必将RUN apt-get update与apt-get install组合在同一个RUN语句中。例如：

```
RUN apt-get update && apt-get install -y \
        package-bar \
        package-baz \
        package-foo
```
        
如果在RUN语句中单独使用apt-get update会导致缓存问题和随后的apt-get install说明失败。例如，说你有一个Docker文件：

```
FROM ubuntu:14.04
RUN apt-get update
RUN apt-get install -y curl
```

构建image后，所有图层都在Docker缓存中。假设你以后通过添加额外的包来修改apt-get install：

```
FROM ubuntu:14.04
RUN apt-get update
RUN apt-get install -y curl nginx
```

Docker将初始和修改的指令看作是相同的，并重新使用先前步骤的缓存。因此，apt-get update不会执行，因为构建使用缓存版本。因为apt-get update没有运行，你的构建可能会有一个过时的curl和nginx包版本。

使用RUN apt-get update && apt-get install -y可确保你的Dockerfile安装最新的软件包版本，无需进一步的编码或手动干预。这种技术被称为“缓存破解”。你还可以通过指定包版本来实现缓存清除。这被称为版本固定，例如：

```
RUN apt-get update && apt-get install -y \
        package-bar \
        package-baz \
        package-foo=1.3.*
```

版本锁定强制构建检索特定版本，而不管缓存中有什么。这种技术还可以减少由于所需软件包中意外的更改导致的故障。

如果image以前使用过旧版本，则指定新版本会导致apt-get update的缓存破坏，并确保新版本的安装。在每行上列出包也可以防止包重复中的错误。

下面是一个完整的运行指令，显示所有apt-get建议。

```
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```

另外，可以通过删除/var/lib/apt/lists来清理apt缓存，减少了image大小，因为apt缓存不存储在图层中。由于RUN语句以apt-get update开头，所以在apt-get install之前，包缓存将始终被刷新。

注意：Debian和Ubuntu的图像自动运行apt-get clean，所以不需要显式调用。

#### 使用管道pipes

一些RUN命令取决于使用管道字符（|）将一个命令的输出管道到另一个命令的能力，如以下示例所示：

```
RUN wget -O - https://some.site | wc -l > /number
```

Docker使用/bin/sh -c解释器执行这些命令，该解释器仅评估管道中最后一个操作的退出代码以确定成功。在上面的示例中，只要wc -l命令成功，即使wget命令失败，构建步骤也会成功并生成新映像。

如果你希望命令由于管道中任何阶段的错误而失败，请先设置-o pipefail &&以确保意外的错误会阻止构建无意中成功。例如：

```
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

注意：并非所有的shell都支持-o pipefail选项。在这种情况下（例如，破折号shell，它是基于Debian的映像的默认shell），请考虑使用exec的形式来显式选择一个支持pipefail选项的shell。例如：

```
RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
```

#### CMD

CMD指令应用于运行image中包含的软件以及任何参数。 CMD几乎总是以CMD [“executable”, “param1”, “param2”…]的形式使用。 因此，如果image用于服务，例如Apache和Rails，则可以运行类似于CMD ["apache2","-DFOREGROUND"]的内容。 实际上，这种形式的指令是推荐用于任何基于服务的image。

在大多数其他情况下，应该给CMD一个交互式的shell，比如bash，python和perl。 例如，CMD ["perl", "-de0"], CMD ["python"], or CMD ["php", "-a"]。 使用这个表单意味着当你执行像docker run -it python时，你将被丢弃到一个可用的shell中。 CMD应该很少以CMD ["param", "param"]的方式与ENTRYPOINT一起使用，除非你和你的用户已经非常熟悉ENTRYPOINT是如何工作的。

#### EXPOSE

EXPOSE指令指示容器将侦听连接的端口。 因此，你应该为应用程序使用通用的传统端口。 例如，包含Apache Web服务器的映像将使用EXPOSE 80，而包含MongoDB的映像将使用EXPOSE 27017等。

对于外部访问，你的用户可以使用指示如何将指定端口映射到所选端口的标志来执行docker运行。 对于容器链接，Docker提供环境变量（例如：MYSQL_PORT_3306_TCP）从目标容器到源容器的路径。

#### ENV

为了使新软件更容易运行，可以为你容器安装的软件使用ENV更新PATH环境变量。 例如，ENV PATH /usr/local/nginx/bin:$PATH将确保CMD ["nginx"]正常工作。

ENV指令也可用于提供特定于要集中化的服务的必需环境变量，例如Postgres的PGDATA。

最后，ENV也可用于设置常用的版本号，以便版本颠覆更容易维护，如下例所示：

```
ENV PG_MAJOR 9.3
ENV PG_VERSION 9.3.4
RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC / usr / src / postgress && ...
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
```

类似于在程序中具有常量变量（与硬编码值相反），这种方法允许你修改单个ENV就能自动控制容器中的软件版本。

#### ADD or COPY

虽然ADD和COPY在功能上是相似的，但一般来说，COPY是首选的。这是因为它比ADD更透明。 COPY只支持将本地文件复制到容器中，而ADD具有一些不是很明显的功能（如本地的tar提取和远程URL支持）。因此，ADD的最佳用途是将本地tar文件自动提取到图像中，如：ADD rootfs.tar.xz /。

如果你有多个Dockerfile步骤可以使用上下文中的不同文件，单独COPY，而不是一次性复制全部文件。如果特定需要的文件更改，这将确保每一步的构建缓存仅被无效（强制该步骤重新运行）。

例如：

```
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

结果就是RUN这步很少的缓存会失效，和把COPY . /tmp/放在RUN之前相比。

由于image大小很重要，因此使用ADD从远程URL获取包是非常不鼓励的，你应该使用curl或wget来代替。这样，你可以删除在解压后不再需要的文件，而不必在image中添加另一个图层。例如，你应该避免这样做：

```
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

而应该这样

```
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

对于不需要ADD tar自动提取功能的其他项目（文件，目录），应始终使用COPY。

#### ENTRYPOINT

ENTRYPOINT的最佳用途是设置image的主命令，允许该image像该命令一样运行（然后使用CMD作为默认标志）。

我们从一个命令行工具s3cmd的图像的例子开始：

```
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

现在可以像这样运行映像来显示命令的帮助：

```
$ docker run s3cmd
```

或使用正确的参数执行命令：

```
$ docker run s3cmd ls s3://mybucket
```

这是有用的，因为image名称可以作为二进制文件的参考，如上面的命令所示。

ENTRYPOINT指令也可以与辅助脚本组合使用，允许其以类似于上述命令的方式运行，即使启动工具可能需要多于一个步骤。

例如，Postgres Official Image使用以下脚本作为其ENTRYPOINT

```
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

注意：此脚本使用exec Bash命令，以便最终运行的应用程序成为容器的PID 1。这允许应用程序接收发送到容器的任何Unix信号。有关详细信息，请参阅ENTRYPOINT帮助。

帮助脚本被复制到容器中，并通过容器起始处的ENTRYPOINT运行：

```
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

此脚本允许用户以多种方式与Postgres进行交互。

它可以简单地启动Postgres：

```
$ docker run postgres
```

或者，它可以用于运行Postgres并将参数--help传递给服务器：

```
$ docker run postgres postgres --help
```

最后，它也可以用来启动一个完全不同的工具，比如Bash：

```
$ docker run --rm -it postgres bash
```

#### Volume

应该使用VOLUME指令来暴露由docker容器创建的任何数据库存储区域，配置存储器或文件/文件夹。

#### User

如果服务可以无特权运行，请使用USER更改为非root用户。 可以使用RUN groupadd -r postgres && useradd -r -g postgres postgres可以创建一个普通用户。

注意：image中的用户和组获得非确定性的UID/GID，因为"next"UID/GID被分配，而不管image重建。 所以，如果是至关重要的，你应该分配一个显式的UID/GID。

你应避免安装或使用sudo，因为它具有不可预测的TTY和信号转发行为，可能导致比解决问题更多的问题。 如果你绝对需要类似于sudo的功能（例如，以root用户身份初始化守护程序，但以非root身份运行），则可以使用"gosu"。

最后，为了降低层次和复杂性，请避免频繁地切换USER。

#### WORKDIR

为了清晰可靠，你应该始终为WORKDIR使用绝对路径。 此外，应该使用WORKDIR，而不应该使用像RUN CD ... && do-something这些难以阅读，排除故障和维护的指令。

#### ONBUILD

在当前的Dockerfile构建完成之后执行一个ONBUILD命令。 ONBUILD在从当前image派生的任何子image中执行。将ONBUILD命令视为父Dockerfile为子Dockerfile提供的指令。

Docker构建在子Dockerfile中的任何命令之前执行ONBUILD命令。

ONBUILD对于那些给定FROM的image构建是很有用的。例如，你可以使用ONBUILD作为语言堆栈image，可以在Dockerfile中构建用该语言编写的任意软件，就像在Ruby的ONBUILD变体中所看到的那样。

从ONBUILD构建的image应该有一个单独的标签，例如：ruby：1.9-onbuild或ruby：2.0-onbuild。

将ADD或COPY放在ONBUILD中时要小心。如果新版本的上下文缺少添加的资源，"ONBUILD"image将会失败。

### 其他建议汇总

#### 移除构建依赖

其实官网的建议中也提到了，只是没有特别的强调。如果通过源码编译构建，你的镜像通常比需要的大很多。可能的话，在同一条RUN指令中，安装构建工具、构建软件，然后移除构建工具。这样可以减少image的大小。

#### 选择gosu

gosu实用工具，通常用在ENTRYPOINT指令调用的脚本中，这些ENTRYPOINT指令位于官方镜像的Dockerfile中。它是个类sudo的简单工具，接受并运行特定用户的特定指令。但是gosu可以避免sudo怪异恼人的TTY和信号转发(signal-forwarding)行为。

#### 不要在 Dockerfile 中修改文件的权限

因为 docker 镜像是分层的，任何修改都会新增一个层，修改文件或者目录权限也是如此。如果修改大文件或者目录的权限，会把这些文件复制一份，这样很容易导致镜像很大。

解决方案也很简单，要么在添加到 Dockerfile 之前就把文件的权限和用户设置好，要么在容器启动脚本（entrypoint）做这些修改。

这里我也是参考了DockerHub上一些官方镜像的写法。

#### apt-get注意点

一个是运行apt-get upgrade 会更新所有包到最新版本 —— 不能这样做的理由是它会妨碍Dockerfile构建的持久与一致性。

另一个是在不同的行之间运行apt-get update与apt-get install命令。不能这样做的原因是，只有apt-get update的代码会在构建过程中被缓存，而且你需要运行apt-get install命令的时候不会每次都被执行。因此，你需要将apt-get update跟所要安装的包都在同一行执行，来确保它们正确的更新。

#### 使用docker exec而不是sshd

需要进入容器要使用docker exec命令，而不要单独安装sshd

参考文章：

- https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/
- http://www.cnblogs.com/vikings-blog/p/4337152.html
- http://www.oschina.net/translate/6-dockerfile-tips-official-images
- http://cizixs.com/2017/03/28/dockerfile-best-practice
