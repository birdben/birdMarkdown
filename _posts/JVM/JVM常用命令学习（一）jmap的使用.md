---
title: "JVM常用命令学习（一）jmap的使用"
date: 2017-02-09 12:59:57
tags: [JVM]
categories: [Java]
---

### Java环境说明

注意：不同版本的JDK可能略有差异

```
$ java -versionjava version "1.7.0_79"Java(TM) SE Runtime Environment (build 1.7.0_79-b15)Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
```

### jmap命令

jmap (Java Memory Map) 用于生成虚拟机的内存快照信息。说白了就是根据Java进程（pid）打印出这个Java进程内存的所有'对象'的情况（如：产生哪些对象，及其数量）

```
$ jmap -helpUsage:    jmap [option] <pid>        (to connect to running process)    jmap [option] <executable <core>        (to connect to a core file)    jmap [option] [server_id@]<remote server IP or hostname>        (to connect to remote debug server)where <option> is one of:    <none>               to print same info as Solaris pmap    -heap                to print java heap summary    -histo[:live]        to print histogram of java object heap; if the "live"                         suboption is specified, only count live objects    -permstat            to print permanent generation statistics    -finalizerinfo       to print information on objects awaiting finalization    -dump:<dump-options> to dump java heap in hprof binary format                         dump-options:                           live         dump only live objects; if not specified,                                        all objects in the heap are dumped.                           format=b     binary format                           file=<file>  dump heap to <file>                         Example: jmap -dump:live,format=b,file=heap.bin <pid>    -F                   force. Use with -dump:<dump-options> <pid> or -histo                         to force a heap dump or histogram when <pid> does not                         respond. The "live" suboption is not supported                         in this mode.    -h | -help           to print this help message    -J<flag>             to pass <flag> directly to the runtime system
```

通过help提示可以看出基本的命令格式有三种

- jmap [option] <pid> : 用于连接一个正在运行的进程
- jmap [option] <executable <core> : 用于连接一个正在运行的进程- jmap [option] [server_id@]<remote server IP or hostname> : 用于连接一个远程server

```
[option]参数：

- core : 将被打印信息的core dump文件
- remote-hostname-or-IP : 远程debug服务的主机名或ip
- server-id : 唯一id，假如一台主机上多个远程debug服务

- <none> : 和命令行使用pmap的效果一样
- dump:[live,]format=b,file=<filename> : 使用hprof二进制形式，输出JVM的heap内容到文件。live子选项是可选的，假如指定live选项,那么只输出活的对象到文件。
- finalizerinfo : 获取正等候回收的对象的信息
- heap : 获取JVM的heap信息，GC使用的算法，heap的配置及wise heap的使用情况。
- histo[:live] : 获取JVM每个class的实例数目，内存占用，类全名信息。JVM的内部类名字开头会加上前缀"*"。如果live子参数加上后,只统计活的对象数量。
- permstat : 获取classload和JVM heap长久层的信息。包含每个classloader的名字，活泼性，地址，父classloader和加载的class数量。另外，内部String的数量和占用内存数也会打印出来。
- F : 当pid没有响应-dump或者-histo时，可以使用-F强制输出heap信息。live子选项不被支持。
```

### 实例

```
# 先查询对应的进程的pid
$ ps -ef | grep zookeeperroot     13779 23226  0 14:52 pts/11   00:00:00 grep --color=auto zookeeperyunyu    22549  2482  0 12:46 pts/11   00:00:11 /usr/local/java/bin/java -Dzookeeper.log.dir=. -Dzookeeper.root.logger=INFO,CONSOLE -cp /usr/local/zookeeper/bin/../build/classes:/usr/local/zookeeper/bin/../build/lib/*.jar:/usr/local/zookeeper/bin/../lib/slf4j-log4j12-1.6.1.jar:/usr/local/zookeeper/bin/../lib/slf4j-api-1.6.1.jar:/usr/local/zookeeper/bin/../lib/netty-3.7.0.Final.jar:/usr/local/zookeeper/bin/../lib/log4j-1.2.16.jar:/usr/local/zookeeper/bin/../lib/jline-0.9.94.jar:/usr/local/zookeeper/bin/../zookeeper-3.4.8.jar:/usr/local/zookeeper/bin/../src/java/lib/*.jar:/usr/local/zookeeper/bin/../conf: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /usr/local/zookeeper/bin/../conf/zoo.cfg

# 根据进程的pid获取JVM的heap信息
$ jmap -heap 22549Attaching to process ID 22549, please wait...Debugger attached successfully.Server compiler detected.JVM version is 24.79-b02using thread-local object allocation.Parallel GC with 2 thread(s)Heap Configuration:   MinHeapFreeRatio = 0   MaxHeapFreeRatio = 100   MaxHeapSize      = 524288000 (500.0MB)   NewSize          = 1310720 (1.25MB)   MaxNewSize       = 17592186044415 MB   OldSize          = 5439488 (5.1875MB)   NewRatio         = 2   SurvivorRatio    = 8   PermSize         = 21757952 (20.75MB)   MaxPermSize      = 85983232 (82.0MB)   G1HeapRegionSize = 0 (0.0MB)Heap Usage:PS Young GenerationEden Space:   capacity = 8912896 (8.5MB)   used     = 838728 (0.7998733520507812MB)   free     = 8074168 (7.700126647949219MB)   9.410274730009192% usedFrom Space:   capacity = 1048576 (1.0MB)   used     = 1032208 (0.9843902587890625MB)   free     = 16368 (0.0156097412109375MB)   98.43902587890625% usedTo Space:   capacity = 1048576 (1.0MB)   used     = 0 (0.0MB)   free     = 1048576 (1.0MB)   0.0% usedPS Old Generation   capacity = 21495808 (20.5MB)   used     = 1814896 (1.7308197021484375MB)   free     = 19680912 (18.769180297851562MB)   8.44302293730945% usedPS Perm Generation   capacity = 22020096 (21.0MB)   used     = 9014328 (8.596733093261719MB)   free     = 13005768 (12.403266906738281MB)   40.93682425362723% used2705 interned Strings occupying 217112 bytes.

# 根据进程的pid获取JVM每个class的实例数目，内存占用，类全名信息。JVM的内部类名字开头会加上前缀"*"。如果live子参数加上后,只统计活的对象数量。
$ jmap -histo 22549
$ jmap -histo:live 22549
...
 231:             3            144  java.lang.ThreadGroup 232:             6            144  java.util.ArrayDeque 233:             3            144  org.apache.zookeeper.server.quorum.QuorumPeer$QuorumServer 234:             6            144  sun.misc.PerfCounter 235:             9            144  sun.reflect.DelegatingMethodAccessorImpl 236:             1            144  org.apache.zookeeper.server.SyncRequestProcessor 237:             3            144  java.net.SocketInputStream 238:             6            144  java.lang.Thread$State 239:             1            136  org.apache.zookeeper.server.quorum.QuorumCnxManager$RecvWorker 240:             1            136  org.apache.zookeeper.server.quorum.CommitProcessor 241:             1            136  org.apache.zookeeper.server.quorum.QuorumCnxManager$SendWorker 242:             4            128  com.sun.jmx.mbeanserver.DefaultMXBeanMappingFactory$ArrayMapping 243:             8            128  java.net.InetSocketAddress 244:             4            128  java.lang.StringCoding$StringEncoder ...

# 获取正等候回收的对象的信息
$ jmap -finalizerinfo 22549Attaching to process ID 22549, please wait...Debugger attached successfully.Server compiler detected.JVM version is 24.79-b02Number of objects pending for finalization: 0

# 获取classload和JVM heap长久层的信息。包含每个classloader的名字，活泼性，地址，父classloader和加载的class数量。
$ jmap -permstat 22549Attaching to process ID 22549, please wait...Debugger attached successfully.Server compiler detected.JVM version is 24.79-b02finding class loader instances ..done.computing per loader stat ..done.please wait.. computing liveness............................................liveness analysis may be inaccurate ...class_loader	classes	bytes	parent_loader	alive?	type<bootstrap>	1282	7414688	  null  	live	<internal>0x00000000e0c0c9e8	1	1888	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000dba50b580x00000000e0c06318	4	13776	  null  	dead	javax/management/remote/rmi/NoCallStackClassLoader@0x00000000dbf4e6900x00000000e0c0c9a8	1	1888	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000dba50b580x00000000e0c07108	2	32144	  null  	dead	javax/management/remote/rmi/NoCallStackClassLoader@0x00000000dbf4e6900x00000000e0c76250	0	0	  null  	live	sun/misc/Launcher$ExtClassLoader@0x00000000dbbc0a480x00000000e0c1b828	0	0	0x00000000e0c76200	dead	java/util/ResourceBundle$RBClassLoader@0x00000000dc0db2300x00000000e0c0c968	1	1888	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000dba50b580x00000000e0c0ca28	1	1888	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000dba50b580x00000000e0c0c928	1	3056	  null  	dead	sun/reflect/DelegatingClassLoader@0x00000000dba50b580x00000000e0c76200	420	2789672	0x00000000e0c76250	live	sun/misc/Launcher$AppClassLoader@0x00000000dbc1b290total = 11	1713	10260888	    N/A    	alive=3, dead=8	    N/A

# 输出JVM的heap内容到文件。
$ jmap -dump:live,format=b,file=heap.bin 22549Dumping heap to /data/logstash-indexer-2.3.4/heap.bin ...Heap dump file created
```

### 遇到问题和解决方法

- Can't attach to the process问题

需要使用root用户来执行jmap命令，否则就必须使用对应启动JVM的用户来执行jmap命令

注意：使用root用户时需要设置Java相关环境变量

```
$ jmap -heap 22549Attaching to process ID 22549, please wait...Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process
```

- Unable to open socket file: target process not responding or HotSpot VM not loaded

具体原因是：

jdk16_21/24开始，JVM启动时产生进程号的临时文件目录优先使用-Djava.io.tmpdir指定的目录，没有指定-Djava.io.tmpdir参数才使用/tmp/hsperfdata_$USER。这里我并没有设置-Djava.io.tmpdir指定的目录，查看/tmp/hsperfdata_$USER目录下的文件，发现zookeeper的进程号22549的pid文件在/tmp/hsperfdata_yunyu目录下，我这里使用root用户执行jmap命令，所以读取不到pid信息才报错。

我这里zookeeper使用的是yunyu用户启动的，所以这里不能使用root用户执行jmap命令，否则就会报下面的错误，需要切换到启动zookeeper的yunyu用户来执行jmap命令。

```
$ ll /tmp/hsperfdata_*/tmp/hsperfdata_root:total 8drwxr-xr-x  2 root root 4096 Feb  9 15:29 ./drwxrwxrwt 12 root root 4096 Feb  9 15:44 ..//tmp/hsperfdata_yunyu:total 40drwxr-xr-x  2 yunyu yunyu  4096 Feb  9 15:25 ./drwxrwxrwt 12 root  root   4096 Feb  9 15:44 ../-rw-------  1 yunyu yunyu 32768 Feb  9 15:44 22549

$ jmap -histo 2254922549: Unable to open socket file: target process not responding or HotSpot VM not loadedThe -F option can be used when the target process is not responding
```

参考文章：

- http://blog.csdn.net/fenglibing/article/details/6411953
- http://stackoverflow.com/questions/2913948/jmap-cant-connect-to-make-a-dump
- http://windows9834.blog.163.com/blog/static/273450042012919101435608/