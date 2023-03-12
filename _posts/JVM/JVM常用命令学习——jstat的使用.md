---
title: "JVM常用命令学习——jstat的使用"
date: 2017-02-09 23:18:01
tags: [JVM]
categories: [Java]
---

### jstat命令

jstat(Java Virtual Machine Statistics Monitoring Tool)。jstat用于监控基于HotSpot的JVM，对其堆的使用情况进行实时的命令行的统计，使用jstat我们可以对指定的JVM做如下监控：

- 类的加载及卸载情况
- 查看新生代、老生代及持久代的容量及使用情况
- 查看新生代、老生代及持久代的垃圾收集情况，包括垃圾回收的次数及垃圾回收所占用的时间
- 查看新生代中Eden区及Survior区中容量及分配情况等

```
$ jstat -help
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as 
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
```

通过help提示可以看出基本的命令格式

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]] : 根据jstat统计的维度不同，可以使用如下表中的选项进行不同维度的统计

参数：

-h<lines> : 用于指定每隔几行就输出列头，如果不指定，默认是只在第一行出现列头。
-J<flag> : 用于将给定的flag传给java应用程序加载器，例如，“-J-Xms48m”将把启动内存设置为48M。如果想查看可以传递哪些选项到应用程序加载器中
-t : 用于在输出内容的第一列显示时间戳，这个时间戳代表的时JVM开始启动到现在的时间（注：在IBM JDK5中是没有这个选项的）。
<vmid> : VM的进程号，即当前运行的java进程号。
<interval> : 间隔时间，单位可以是秒或者毫秒，通过指定s或ms确定，默认单位为毫秒。
<count> : 打印次数，如果缺省则打印无数次。

[option]参数：

-class : 用于查看类加载情况的统计
-compiler : 用于查看HotSpot中即时编译器编译情况的统计
-gc : 用于查看JVM中堆的垃圾收集情况的统计
-gccapacity : 用于查看新生代、老生代及持久代的存储容量情况
-gccause : 用于查看垃圾收集的统计情况（这个和-gcutil选项一样），如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因。
-gcnew : 用于查看新生代垃圾收集的情况
-gcnewcapacity : 用于查看新生代的存储容量情况
-gcold : 用于查看老生代及持久代发生GC的情况
-gcoldcapacity : 用于查看老生代的容量
-gcpermcapacity : 用于查看持久代的容量（JDK7使用）
-gcmetavcapacity : 用于查看元空间的容量（JDK8使用）
-gcutil : 用于查看新生代、老生代及持代垃圾收集的情况
-printcompilation : HotSpot编译方法的统计
```


-class : 类加载情况的统计

列名		|说明
---			|---
Loaded		|加载了的类的数量
Bytes		|加载了的类的大小，单为Kb
Unloaded	|卸载了的类的数量
Bytes		|卸载了的类的大小，单为Kb
Time		|花在类的加载及卸载的时间
     
-compiler : HotSpot中即时编译器编译情况的统计

列名			|说明
---				|---
Compiled		|编译任务执行的次数
Failed			|编译任务执行失败的次数
Invalid		|编译任务非法执行的次数
Time			|执行编译花费的时间
FailedType	|最后一次编译失败的编译类型
FailedMethod	|最后一次编译失败的类名及方法名

-gc : JVM中堆的垃圾收集情况的统计

列名		|说明
---			|---
S0C			|新生代中Survivor space中S0当前容量的大小（KB）
S1C			|新生代中Survivor space中S1当前容量的大小（KB）
S0U			|新生代中Survivor space中S0容量使用的大小（KB）
S1U			|新生代中Survivor space中S1容量使用的大小（KB）
EC			|Eden space当前容量的大小（KB）
EU			|Eden space容量使用的大小（KB）
OC			|Old space当前容量的大小（KB）
OU			|Old space使用容量的大小（KB）
PC			|Permanent space当前容量的大小（KB）（JDK7使用）
PU			|Permanent space使用容量的大小（KB）（JDK7使用）
MC			|Meta space当前容量的大小（KB）（JDK8使用）
MU			|Meta space使用容量的大小（KB）（JDK8使用）
CCSC		|压缩类空间大小（JDK8使用）
CCSU		|压缩类空间使用大小（JDK8使用）
YGC			|从应用程序启动到采样时发生 Young GC 的次数
YGCT		|从应用程序启动到采样时 Young GC 所用的时间(秒)
FGC			|从应用程序启动到采样时发生 Full GC 的次数
FGCT		|从应用程序启动到采样时 Full GC 所用的时间(秒)
GCT			|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-gccapacity : 新生代、老生代及持久代的存储容量情况

列名	|说明
---		|---
NGCMN	|新生代的最小容量大小（KB）
NGCMX	|新生代的最大容量大小（KB）
NGC		|当前新生代的容量大小（KB）
S0C		|当前新生代中survivor space 0的容量大小（KB）
S1C		|当前新生代中survivor space 1的容量大小（KB）
EC		|Eden space当前容量的大小（KB）
OGCMN	|老生代的最小容量大小（KB）
OGCMX	|老生代的最大容量大小（KB）
OGC		|当前老生代的容量大小（KB）
OC		|当前老生代的空间容量大小（KB）
PGCMN	|持久代的最小容量大小（KB）（JDK7使用）
PGCMX	|持久代的最大容量大小（KB）（JDK7使用）
PGC		|当前持久代的容量大小（KB）（JDK7使用）
PC		|当前持久代的空间容量大小（KB）（JDK7使用）
MCMN	|元空间的最小容量大小（KB）（JDK8使用）
MCMX	|元空间的最大容量大小（KB）（JDK8使用）
MC		|当前元空间的容量大小（KB）（JDK8使用）
CCSMN	|压缩类空间的最小容量大小（KB）（JDK8使用）
CCSMX	|压缩类空间的最大容量大小（KB）（JDK8使用）
CCSC	|当前压缩类空间的容量大小（KB）（JDK8使用）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
FGC		|从应用程序启动到采样时发生 Full GC 的次数

-gccause : 用于查看垃圾收集的统计情况，包括最近发生垃圾的原因

这个选项用于查看垃圾收集的统计情况（这个和-gcutil选项一样），如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因，它比-gcutil会多出最后一次垃圾收集原因以及当前正在发生的垃圾收集的原因。

列名	|说明
---		|---
LGCC	|最后一次垃圾收集的原因，可能为“unknown GCCause”、“System.gc()”等
GCC		|当前垃圾收集的原因

-gcnew : 新生代垃圾收集的情况

列名	|说明
---		|---
S0C		|当前新生代中survivor space 0的容量大小（KB）
S1C		|当前新生代中survivor space 1的容量大小（KB）
S0U		|S0已经使用的大小（KB）
S1U		|S1已经使用的大小（KB）
TT		|Tenuring threshold，要了解这个参数，我们需要了解一点Java内存对象的结构，在Sun JVM中，（除了数组之外的）对象都有两个机器字（words）的头部。第一个字中包含这个对象的标示哈希码以及其他一些类似锁状态和等标识信息，第二个字中包含一个指向对象的类的引用，其中第二个字节就会被垃圾收集算法使用到。在新生代中做垃圾收集的时候，每次复制一个对象后，将增加这个对象的收集计数，当一个对象在新生代中被复制了一定次数后，该算法即判定该对象是长周期的对象，把他移动到老生代，这个阈值叫着tenuring threshold。这个阈值用于表示某个/些在执行批定次数youngGC后还活着的对象，即使此时新生的的Survior没有满，也同样被认为是长周期对象，将会被移到老生代中。
MTT		|Maximum tenuring threshold，用于表示TT的最大值。
DSS		|Desired survivor size (KB).可以参与这里：http://blog.csdn.net/yangjun2/article/details/6542357
EC		|Eden space当前容量的大小（KB）
EU		|Eden space已经使用的大小（KB）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
YGCT	|从应用程序启动到采样时 Young GC 所用的时间(单位秒)


-gcnewcapacity : 新生代的存储容量情况

列名	|说明
---		|---
NGCMN	|新生代的最小容量大小（KB）
NGCMX	|新生代的最大容量大小（KB）
NGC		|当前新生代的容量大小（KB）
S0CMX	|新生代中SO的最大容量大小（KB）
S0C		|当前新生代中SO的容量大小（KB）
S1CMX	|新生代中S1的最大容量大小（KB）
S1C		|当前新生代中S1的容量大小（KB）
ECMX	|新生代中Eden的最大容量大小（KB）
EC		|当前新生代中Eden的容量大小（KB）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
FGC		|从应用程序启动到采样时发生 Full GC 的次数

-gcold : 老生代及持久代发生GC的情况

列名	|说明
---		|---
PC		|Permanent space当前容量的大小（KB）（JDK7使用）
PU		|Permanent space使用容量的大小（KB）（JDK7使用）
MC		|Meta space当前容量的大小（KB）（JDK8使用）
MU		|Meta space使用容量的大小（KB）（JDK8使用）
CCSC	|压缩类空间大小（JDK8使用）
CCSU	|压缩类空间使用大小（JDK8使用）
OC		|当前老年代容量的大小（KB）
OU		|老年代使用容量的大小（KB）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
FGC		|从应用程序启动到采样时发生 Full GC 的次数
FGCT	|从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT		|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-gcoldcapacity : 老生代的存储容量情况

列名	|说明
---		|---
OGCMN	|老生代的最小容量大小（KB）
OGCMX	|老生代的最大容量大小（KB）
OGC		|当前老生代的容量大小（KB）
OC		|当前新生代的空间容量大小（KB）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
FGC		|从应用程序启动到采样时发生 Full GC 的次数
FGCT	|从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT		|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-gcpermcapacity : 持久代的存储容量情况（JDK8已经换成gcmetacapacity）

列名	|说明
---		|---
PGCMN	|持久代的最小容量大小（KB）
PGCMX	|持久代的最大容量大小（KB）
PGC		|当前持久代的容量大小（KB）
PC		|当前持久代的空间容量大小（KB）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
FGC		|从应用程序启动到采样时发生 Full GC 的次数
FGCT	|从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT		|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-gcmetacapacity : 元空间的存储容量情况（替换JDK7的gcpermcapacity）

列名		|说明
---			|---
MCMN		|元空间的最小容量大小（KB）
MCMX		|元空间的最大容量大小（KB）
MC			|当前元空间的容量大小（KB）
CCSMN		|压缩类空间的最小容量大小（KB）
CCSMX		|压缩类空间的最大容量大小（KB）
CCSC		|当前压缩类空间的容量大小（KB）
YGC			|从应用程序启动到采样时发生 Young GC 的次数
FGC			|从应用程序启动到采样时发生 Full GC 的次数
FGCT		|从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT			|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-gcutil : 新生代、老生代及持代垃圾收集的情况

列名	|说明
---		|---
S0		|Heap上的 Survivor space 0 区已使用空间的百分比
S1		|Heap上的 Survivor space 1 区已使用空间的百分比
E		|Heap上的 Eden space 区已使用空间的百分比
O		|Heap上的 Old space 区已使用空间的百分比
P		|Perm space 区已使用空间的百分比（JDK7使用）
M		|Meta space 区已使用空间的百分比（JDK8使用）
CCS		|压缩类已使用空间的百分比（JDK8使用）
YGC		|从应用程序启动到采样时发生 Young GC 的次数
YGCT	|从应用程序启动到采样时 Young GC 所用的时间(单位秒)
FGC		|从应用程序启动到采样时发生 Full GC 的次数
FGCT	|从应用程序启动到采样时 Full GC 所用的时间(单位秒)
GCT		|从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC

-printcompilation : HotSpot编译方法的统计

列名		|说明
---			|---
Compiled	|编译任务执行的次数
Size		|方法的字节码所占的字节数
Type		|编译类型
Method		|指定确定被编译方法的类名及方法名，类名中使名“/”而不是“.”做为命名分隔符，方法名是被指定的类中的方法，这两个字段的格式是由HotSpot中的“-XX:+PrintComplation”选项确定的。

### 实例

```
$ jstat -class 2699
Loaded  Bytes  Unloaded  Bytes     Time
  3226  6180.9        0     0.0       1.39

$ jstat -compiler 2699
Compiled Failed Invalid   Time   FailedType FailedMethod
    2710      0       0     5.04          0
  
$ jstat -gc 2699
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
11776.0 18944.0 11560.9  0.0   126976.0  2297.8   175104.0   14553.3   19840.0 19174.8 2432.0 2175.6      6    0.183   0      0.000    0.183
     
$ jstat -gccapacity 2699
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC
 87040.0 1397760.0 300032.0 18944.0 10752.0 131072.0   175104.0  2796544.0   175104.0   175104.0      0.0 1067008.0  19840.0      0.0 1048576.0   2432.0      5     0
 
$ jstat -gccause 2699
 S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
 0.00  99.93  98.25   8.31  96.64  89.44      5    0.144     0    0.000    0.144 Allocation Failure   No GC
 
$ jstat -gcnew 2699
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
18944.0 10752.0    0.0 10744.9  6  15 18944.0 131072.0 128773.2      5    0.144

$ jstat -gcnewcapacity 2699
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
   87040.0  1397760.0   300032.0 465920.0  18944.0 465920.0  10752.0  1396736.0   131072.0     5     0

$ jstat -gcold 2699
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
 19840.0  19173.1   2432.0   2175.1    175104.0     14545.3      5     0    0.000    0.144
 
$ jstat -gcoldcapacity 2699
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
   175104.0   2796544.0    175104.0    175104.0     5     0    0.000    0.144

# 这里jdk8已经不是gcpermcapacity了，换成gcmetacapacity
$ jstat -gcmetacapacity 2699
   MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT
       0.0  1067008.0    19840.0        0.0  1048576.0     2432.0     5     0    0.000    0.144

$ jstat -gcutil 2699
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  99.93  98.25   8.31  96.64  89.44      5    0.144     0    0.000    0.144

$ jstat -printcompilation 2699
Compiled  Size  Type Method
    2709   1871    1 sun/net/spi/DefaultProxySelector$3 run
```

测试其他参数

```
# 每隔1秒输出一次
$ jstat -gcutil 2699 1s
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 98.17   0.00   5.23   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   5.23   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   5.23   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   5.23   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   5.23   8.31  96.65  89.46      6    0.183     0    0.000    0.183

# 每隔2行显示一次列头
$ jstat -gcutil -h 2 2699 1s
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
 
# 在第一列显示时间戳
$ jstat -gcutil -t -h 2 2699 1s
Timestamp         S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
        85813.0  98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
        85814.0  98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
Timestamp         S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
        85815.0  98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
        85816.0  98.17   0.00   6.94   8.31  96.65  89.46      6    0.183     0    0.000    0.183
```

> 注意：
>
> 以上命令都在JDK8环境下执行（和JDK7略有不同）

参考文章：

- http://www.softown.cn/post/187.html
- http://blog.csdn.net/fenglibing/article/details/6411951
