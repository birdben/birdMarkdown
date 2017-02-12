---
title: "VisualVM.md"
date: 2016-08-20 23:25:38
tags: [JVM]
categories: [Java]
---

### 使用jstatd方式远程监控Linux下的JVM运行情况

在${JAVA_HOME}/bin目录下创建jstatd.all.policy安全策略文件

```
grant codebase "file:${java.home}/../lib/tools.jar" {
    permission java.security.AllPermission;
};
```

#### jstatd 命令

```
jstatd 命令

options
-nr : 当一个存在的RMI Registry没有找到时，不尝试创建一个内部的RMI Registry
-p : port 端口号，默认为1099
-n : rminame 默认为JStatRemoteHost；如果多个jstatd服务开始在同一台主机上，rminame唯一确定一个jstatd服务
-J : jvm选项

$ cd $JAVA_HOME/bin

# 启动jstatd服务，使用内部RMI Registry默认端口号1099
$ ./jstatd -J-Djava.security.policy=jstatd.all.policy

# 注册一个RMI端口
$ rmiregistry 2020&
# 启动jstatd服务，指定安全策略文件，使用外部RMI Registry指定端口号2020
$ ./jstatd -J-Djava.security.policy=jstatd.all.policy -J-Djava.rmi.server.hostname=192.168.1.100 -p 2020
```


若出现下面的问题，是因为没有给jstatd指定安全策略，如上新建安全策略文件后运行指定文件即可

```
Could not create remote object  
access denied (java.util.PropertyPermission java.rmi.server.ignoreSubClasses write)  
java.security.AccessControlException: access denied (java.util.PropertyPermission java.rmi.server.ignoreSubClasses write)  
        at java.security.AccessControlContext.checkPermission(AccessControlContext.java:323)  
        at java.security.AccessController.checkPermission(AccessController.java:546)  
        at java.lang.SecurityManager.checkPermission(SecurityManager.java:532)  
        at java.lang.System.setProperty(System.java:725)  
        at sun.tools.jstatd.Jstatd.main(Jstatd.java:122)  
```

```
# 下载并启动VisualVM
$ ./visualvm --jdkhome $JAVA_HOME --userdir ~/
```

启动成功后，新建一个Remote Host（指定你要监控的主机IP地址），就可以看到对应的使用JVM的进程以及PID了，还可以查看具体进程的CPU，Heap，Threads等情况。

![VisualVM](http://img.blog.csdn.net/20161216122534885?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![VisualVM_Monitor](http://img.blog.csdn.net/20161216133120180?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### JMX方式监控

这里使用Kafka举例，Kafka启动的时候指定开启的JMX端口9999

```
$ JMX_PORT=9999 ./kafka-server-start.sh ../config/server.properties &
```

然后使用VisualVM工具监控Kafka，先安装MBeans插件

- 添加一个JMX连接

![添加JMX连接](http://img.blog.csdn.net/20170206192807778?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- JMX连接的端口是我们前面启动使用的9999

![JMX连接信息](http://img.blog.csdn.net/20170206192827680?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- MBean插件显示具体的监控信息

![监控详情](http://img.blog.csdn.net/20170206192732559?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

参考文章：

- http://docs.oracle.com/javase/7/docs/technotes/tools/share/jstatd.html
- https://my.oschina.net/u/218540/blog/263704