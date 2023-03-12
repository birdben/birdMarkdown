---
title: "JVM常用命令学习——jstack的使用"
date: 2017-07-30 16:52:11
tags: [JVM]
categories: [Java]
---

### jstack命令

jstack是一个堆栈跟踪工具，主要用于打印指定Java进程、核心文件或远程调试服务器的Java线程的堆栈跟踪信息。

```
$ jstack --help
Usage:
    jstack [-l] <pid>
        (to connect to running process)
    jstack -F [-m] [-l] <pid>
        (to connect to a hung process)
    jstack [-m] [-l] <executable> <core>
        (to connect to a core file)
    jstack [-m] [-l] [server_id@]<remote server IP or hostname>
        (to connect to a remote debug server)

Options:
    -F  to force a thread dump. Use when jstack <pid> does not respond (process is hung)
    -m  to print both java and native frames (mixed mode)
    -l  long listing. Prints additional information about locks
    -h or -help to print this help message
```

通过help提示可以看出基本的命令输入格式为：

```
jstack [-l] <pid> : 指定进程号(pid)的进程jstack -F [-m] [-l] <pid> : pid无法响应时，强制打印堆栈
jstack [-m] [-l] <executable> <core> : 指定核心文件
jstack [-m] [-l] [server_id@]<remote server IP or hostname> : 指定远程调试服务器

参数：

<pid> : 需要打印堆栈信息的进程ID。
<executable> : 产生核心dump的Java可执行文件。
<core> : 需要打印配置信息的核心文件。
<remote-hostname-or-IP/hostname> : 远程调试服务器的主机名或IP地址。
<server-id> : 可选的唯一id，如果相同的远程主机上运行了多台调试服务器，用此选项参数标识服务器。

[option]参数：

<no option> : 打印命令行标识参数和系统属性键值对。
-F : pid无法响应时，强制打印堆栈。
-l : 长列表。打印关于锁的附加信息，例如属于java.util.concurrent的ownable synchronizers列表。
-m : 混合模式输出(包括java和本地c/c++片段)堆栈。
-h : 打印帮助信息。
-help : 打印帮助信息。
```

线程状态：

- RUNNABLE: 运行中状态，可能里面还能看到locked字样，表明它获得了某把锁。
- BLOCKED：被某个锁(synchronizers)給block住了。
- WAITING：等待某个condition或monitor发生，一般停留在park(), wait(), sleep(),join() 等语句里。
- TIME_WAITING：和WAITING的区别是wait() 等语句加上了时间限制 wait(timeout)。

### 实例

待补充

参考文章：

- http://www.softown.cn/post/186.html
