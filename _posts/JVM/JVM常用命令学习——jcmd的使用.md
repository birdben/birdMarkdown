---
title: "JVM常用命令学习——jcmd的使用"
date: 2017-07-30 18:35:08
tags: [JVM]
categories: [Java]
---

### jcmd命令

jcmd用于向正在运行的JVM发送诊断信息请求。

```
$ jcmd -help
Usage: jcmd <pid | main class> <command ...|PerfCounter.print|-f file>
   or: jcmd -l
   or: jcmd -h

  command must be a valid jcmd command for the selected jvm.
  Use the command "help" to see which commands are available.
  If the pid is 0, commands will be sent to all Java processes.
  The main class argument will be used to match (either partially
  or fully) the class used to start Java.
  If no options are given, lists Java processes (same as -p).

  PerfCounter.print display the counters exposed by this process
  -f  read and execute commands from the file
  -l  list JVM processes on the local machine
  -h  this help
```

通过help提示可以看出基本的命令格式

```
jcmd [ options ]
jcmd <pid | main class> PerfCounter.print
jcmd <pid | main class> command <arguments>
jcmd <pid | main class> -f <file>

参数：

<pid> : 接收诊断命令请求的进程ID。该进程必须是一个Java进程。你可以使用jps或jcmd来获取运行于当前计算机上的Java进程列表。
<main-class> : 接收诊断命令请求的进程的main类。匹配进程时，main类名称中包含指定子字符串的任何进程均是匹配的。如果多个正在运行的Java进程共享同一个main类，诊断命令请求将会发送到所有的这些进程中。
command <arguments> : 接收诊断命令请求的进程的main类。匹配进程时，main类名称中包含指定子字符串的任何进程均是匹配的。如果多个正在运行的Java进程共享同一个main类，诊断命令请求将会发送到所有的这些进程中。
注意: 如果任何参数含有空格，你必须使用英文的单引号或双引号将其包围起来。 此外，你必须使用转义字符来转移参数中的单引号或双引号，以阻止操作系统shell处理这些引用标记。当然，你也可以在参数两侧加上单引号，然后在参数内使用双引号(或者，在参数两侧加上双引号，在参数中使用单引号)。
Perfcounter.print : 打印目标Java进程上可用的性能计数器。性能计数器的列表可能会随着Java进程的不同而产生变化。
-f <file> : 从文件file中读取命令，然后在目标Java进程上调用这些命令。在file中，每个命令必须写在单独的一行。以"#"开头的行会被忽略。当所有行的命令被调用完毕后，或者读取到含有stop关键字的命令，将会终止对file的处理。

[option]参数：

-l : 打印正在运行的Java进程的列表信息，包括进程ID、main类以及它们的命令行参数。
-h : 打印帮助信息。
-help : 打印帮助信息。
```

### 实例

```
# 从PerfCounter.print打印出的内容搜索"java.cls.loadedClasses"
$ jcmd 27521 PerfCounter.print | grep "java.cls.loadedClasses"
java.cls.loadedClasses=2548


# 查看帮助信息
$ jcmd 27521 help
27521:
The following commands are available:
JFR.stop
JFR.start
JFR.dump
JFR.check
VM.native_memory
VM.check_commercial_features
VM.unlock_commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
GC.rotate_log
Thread.print
GC.class_stats
GC.class_histogram
GC.heap_dump
GC.run_finalization
GC.run
VM.uptime
VM.flags
VM.system_properties
VM.command_line
VM.version
help

For more information about a specific command use 'help <command>'.


# 查看VM.flags命令帮助信息
$ jcmd 27521 help VM.flags
27521:
VM.flags
Print VM flag options and their current values.

Impact: Low

Permission: java.lang.management.ManagementPermission(monitor)

Syntax : VM.flags [options]

Options: (options must be specified using the <key> or <key>=<value> syntax)
	-all : [optional] Print all flags supported by the VM (BOOLEAN, false)


# 打印VM.flags启动参数信息
$ jcmd 27521 VM.flags
27521:
-XX:CICompilerCount=4 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC


# 导出堆信息到指定文件中
$ jcmd 27521 GC.heap_dump /Users/yunyu/tomcat.dump
27521:
Heap dump file created
```

参考文章：

- http://www.softown.cn/post/178.html