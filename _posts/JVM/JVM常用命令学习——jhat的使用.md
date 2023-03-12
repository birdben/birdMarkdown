---
title: "JVM常用命令学习——jhat的使用"
date: 2017-07-30 18:06:07
tags: [JVM]
categories: [Java]
---

### jhat命令

jhat(Java Heap Analysis Tool)，是JDK自带的Java堆内存分析工具。

```
$ jhat -help
Usage:  jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

	-J<flag>          Pass <flag> directly to the runtime system. For
			  example, -J-mx512m to use a maximum heap size of 512MB
	-stack false:     Turn off tracking object allocation call stack.
	-refs false:      Turn off tracking of references to objects
	-port <port>:     Set the port for the HTTP server.  Defaults to 7000
	-exclude <file>:  Specify a file that lists data members that should
			  be excluded from the reachableFrom query.
	-baseline <file>: Specify a baseline object dump.  Objects in
			  both heap dumps with the same ID and same class will
			  be marked as not being "new".
	-debug <int>:     Set debug level.
			    0:  No debug output
			    1:  Debug hprof file parsing
			    2:  Debug hprof file parsing, no server
	-version          Report version number
	-h|-help          Print this help and exit
	<file>            The file to read

For a dump file that contains multiple heap dumps,
you may specify which dump in the file
by appending "#<number>" to the file name, i.e. "foo.hprof#3".

All boolean options default to "true"
```

通过help提示可以看出基本的命令格式

```
jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

参数：

<file> : 指定用于浏览的Java二进制heap dump文件。对于一个包含多个heap dump的dump文件，你可以在文件名称后面追加"#<number>"来指定文件中的某个dump，例如：foo.hprof#3。

[option]参数：

-stack <bool> : 关闭跟踪对象分配调用堆栈。注意，如果heap dump中的分配位置信息不可用，你必须设置此标识为false。此选项的默认值为true。
-refs <bool> : 关闭对象的引用跟踪。默认为true。默认情况下，反向指针(指向给定对象的对象，又叫做引用或外部引用)用于计算堆中的所有对象.
-port <port> : 设置jhat的HTTP服务器的端口号。默认为7000。
-exclude <file> : 指定一个数据成员列表的文件，这些数据成员将被排除在"reachable objects"查询的范围之外。举个例子，如果文件列有java.lang.String.value，那么，当计算指定对象"o"的可达对象列表时，涉及到java.lang.String.value字段的引用路径将会被忽略掉。
-baseline <file> : 指定一个基线heap dump。在两个heap dump(当前heap dump和基线heap dump)中存在相同对象ID的对象，不会被标记为"new"。其他的对象将被标记为"new"。这在比较两个不同的heap dump时非常有用。
-debug <int> : 设置此工具的调试级别。0意味着没有调试输出。设置的值越高，输出的信息就越详细。
-version : 报告版本号并退出。
-h : 输出帮助信息并退出。
-help : 输出帮助信息并退出。
-J<flag> : 将运行时参数传递给运行jhat的JVM。例如，-J-Xmx512m设置使用的最大堆内存大小为512MB。
```

### 实例

待补充

参考文章：

- http://www.softown.cn/post/181.html