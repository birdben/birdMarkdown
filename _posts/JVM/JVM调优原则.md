---
title: "JVM调优原则"
date: 2017-08-09 18:13:51
tags: [JVM]
categories: [Java]
---

垃圾收集调优基础

怎样监控Java程序？
应该给JVM设置怎样的参数？
如何确定是否需要修改代码？

- 吞吐量
 * TPS: 每秒多少次事务
 * QPS: 每秒多少次查询

- 延迟：比如关键请求必须60ms完成响应

- 内存占用

基本原则

1. 每次MinorGC都尽可能多地收集垃圾对象。可以减少FullGC的频率，因为FullGC的持续时间总是最长；

2. 处理吞吐量和延迟问题时，GC能使用的内存越大，垃圾收集的效果越好，应用越流畅；

3. 在这三个属性（吞吐量、延迟、内存占用）中任意选择两个进行JVM垃圾收集器调优，因为三个属性肯定不能同时满足；


- 线程池：解决用户响应时间长的问题
- 连接池：解决并发问题，提高吞吐
- JVM启动参数：调整各代的内存比例和垃圾回收算法，提高吞吐量
- 程序算法：改进程序逻辑算法提高性能

JVM调优相关

- GC的时间足够的小
- GC的次数足够的少
- 发生Full GC的周期足够的长
- 前两个目前是相悖的，要想GC时间小必须要一个更小的堆，要保证GC次数足够少，必须保证一个更大的堆，我们只能取其平衡。

（1）针对JVM堆的设置一般，可以通过-Xms -Xmx限定其最小、最大值，为了防止垃圾收集器在最小、最大之间收缩堆而产生额外的时间，我们通常把最大、最小设置为相同的值
  
（2）年轻代和年老代将根据默认的比例（1：2）分配堆内存，可以通过调整二者之间的比率NewRadio来调整二者之间的大小，也可以针对回收代，比如年轻代，通过 -XX:newSize -XX:MaxNewSize来设置其绝对大小。同样，为了防止年轻代的堆收缩，我们通常会把-XX:newSize -XX:MaxNewSize设置为同样大小。

（3）年轻代和年老代设置多大才算合理？这个我问题毫无疑问是没有答案的，否则也就不会有调优。我们观察一下二者大小变化有哪些影响

- 更大的年轻代必然导致更小的年老代，大的年轻代会延长普通GC的周期，但会增加每次GC的时间；小的年老代会导致更频繁的Full GC。
- 更小的年轻代必然导致更大年老代，小的年轻代会导致普通GC很频繁，但每次的GC时间会更短；大的年老代会减少Full GC的频率。
- 如何选择应该依赖应用程序对象生命周期的分布情况：如果应用存在大量的临时对象，应该选择更大的年轻代；如果存在相对较多的持久对象，年老代应该适当增大。但很多应用都没有这样明显的特性，在抉择时应该根据以下两点：
 * （A）本着Full GC尽量少的原则，让年老代尽量缓存常用对象，JVM的默认比例1：2也是这个道理
 * （B）通过观察应用一段时间，看其他在峰值时年老代会占多少内存，在不影响Full GC的前提下，根据实际情况加大年轻代，比如可以把比例控制在1：1。但应该给年老代至少预留1/3的增长空间。

（4）在配置较好的机器上（比如多核、大内存），可以为年老代选择并行收集算法：-XX:+UseParallelOldGC，默认为Serial收集

（5）线程堆栈的设置：每个线程默认会开启1M的堆栈，用于存放栈帧、调用参数、局部变量等，对大多数应用而言这个默认值太了，一般256K就足用。理论上，在内存不变的情况下，减少每个线程的堆栈，可以产生更多的线程，但这实际上还受限于操作系统。

（6）可以通过下面的参数打Heap Dump信息

```
-XX:HeapDumpPath
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:/usr/aaa/dump/heap_trace.txt
```

通过下面参数可以控制OutOfMemoryError时打印堆的信息

```
-XX:+HeapDumpOnOutOfMemoryError
```

请看一下一个时间的Java参数配置：（服务器：Linux 64Bit，8Core×16G）

```
JAVA_OPTS="$JAVA_OPTS -server -Xms3G -Xmx3G -Xss256k
-XX:PermSize=128m -XX:MaxPermSize=128m -XX:+UseParallelOldGC
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/aaa/dump
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps
-Xloggc:/usr/aaa/dump/heap_trace.txt -XX:NewSize=1G -XX:MaxNewSize=1G"
```

经过观察该配置非常稳定，每次普通GC的时间在10ms左右，Full GC基本不发生，或隔很长很长的时间才发生一次。
