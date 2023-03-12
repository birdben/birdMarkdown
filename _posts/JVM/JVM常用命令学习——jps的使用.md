---
title: "JVM常用命令学习——jps的使用"
date: 2017-02-09 18:23:28
tags: [JVM]
categories: [Java]
---

### jps命令

jps(Java Virtual Machine Process Status Tool)。jps只是用来显示Java进程信息，用来查看基于HotSpot的JVM里面中，所有具有访问权限的Java进程的具体状态, 包括进程ID，进程启动的路径及启动参数等等。

如果使用jps查看远程服务器的Java进程信息，需要在远程服务器上开启jstatd服务。

```
$ jps -helpusage: jps [-help]       jps [-q] [-mlvV] [<hostid>]Definitions:    <hostid>:      <hostname>[:<port>]
```

通过help提示可以看出基本的命令输入格式为：

```
jps [option] : 查看Java进程信息jps [option] <hostname>[:<port>] : 查看一个远程server的Java进程信息，port是远程rmi的端口，如果没有指定则默认为1099。

参数：
<hostname> : 远程debug服务的主机名或ip<port> : 远程debug服务的端口号

[option]参数：

<no option> : 输出进程的pid和应用程序主类的名称（不是完整包名）。
-q : 只输出进程的pid
-m : 输出传递给main方法的参数，如果是内嵌的JVM则输出为null。
-l : 输出应用程序主类的完整包名，或者是应用程序JAR文件的完整路径。
-v : 输出传给JVM的参数
-V : 输出通过标记的文件传递给JVM的参数（.hotspotrc文件，或者是通过参数-XX:Flags=<filename>指定的文件）。
```

输出格式为：

```
lvmid[ [classname|JARfilename|"Unknown"] [arg* ] [jvmarg* ] ]

lvmid : 进程的pid
[classname|JARfilename|"Unknown"] : 应用程序主类的完整包名，或者是应用程序JAR文件的完整路径。
[arg* ] : 传递给main方法的参数
[jvmarg* ] : 传给JVM的参数
```

### 实例

```
$ jps22549 QuorumPeerMain18187 Jps14983 Jstatd

# 只输出进程的pid
$ jps -q225491498318259

# 输出应用程序主类的完整包名，或者是应用程序JAR文件的完整路径。
$ jps -l22549 org.apache.zookeeper.server.quorum.QuorumPeerMain18113 sun.tools.jps.Jps14983 sun.tools.jstatd.Jstatd

# 输出传递给main方法的参数，如果是内嵌的JVM则输出为null。
$ jps -m22549 QuorumPeerMain /usr/local/zookeeper/bin/../conf/zoo.cfg18344 Jps -m14983 Jstatd

# 输出传给JVM的参数
$ jps -v22549 QuorumPeerMain -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false17992 Jps -Dapplication.home=/data/jdk1.7.0_79 -Xms8m14983 Jstatd -Dapplication.home=/data/jdk1.7.0_79 -Xms8m -Djava.security.policy=jstatd.all.policy

# 在hadoop1的机器的${JAVA_HOME}/bin目录下创建jstatd.all.policy安全策略文件
grant codebase "file:${java.home}/../lib/tools.jar" {
    permission java.security.AllPermission;
};

# 在hadoop1的机器启动jstatd服务，使用内部RMI Registry默认端口号1099
$ ./jstatd -J-Djava.security.policy=jstatd.all.policy

# 在其他机器查看hadoop1的Java进程信息
$ jps -l hadoop1
$ jps -l hadoop1:1099
14983 sun.tools.jstatd.Jstatd
22549 org.apache.zookeeper.server.quorum.QuorumPeerMain
```

参考文章：

- http://www.softown.cn/post/184.html
