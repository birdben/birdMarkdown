---
title: "JVM参数总结"
date: 2017-02-14 01:16:39
tags: [JVM]
categories: [Java]
---

### 

#### JVM参数的含义

参数名称|含义|默认值|备注
---|---|---|---
-Xms|初始堆大小|物理内存的1/64(<1GB)|默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制
-Xmx|最大堆大小|物理内存的1/4(<1GB)|默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制
-Xmn|年轻代大小(1.4or later)	|-|注意：此处的大小是（eden+ 2 survivor space)。与jmap -heap中显示的New gen是不同的。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
-XX:NewSize|设置年轻代大小(for 1.3/1.4)|-|-
-XX:MaxNewSize|年轻代最大值(for 1.3/1.4)|-|-
-XX:PermSize|设置持久代(perm gen)初始值|物理内存的1/64|-
-XX:MaxPermSize|设置持久代最大值|物理内存的1/4|-
-Xss|每个线程的堆栈大小|-|JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。一般小的应用，如果栈不是很深，应该是128k够用的，大的应用建议使用256k。这个选项对性能影响比较大，需要严格的测试。（校长）和threadstacksize选项解释很类似，官方文档似乎没有解释，在论坛中有这样一句话:"-Xss is translated in a VM flag named ThreadStackSize"一般设置这个值就可以了。
-XX:ThreadStackSize|Thread Stack Size|-|(0 means use default stack size) [Sparc: 512; Solaris x86: 320 (was 256 prior in 5.0 and earlier); Sparc 64 bit: 1024; Linux amd64: 1024 (was 0 in 5.0 and earlier); all others 0.]
-XX:NewRatio|年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)|-XX:NewRatio=4表示年轻代与年老代所占比值为1:4,年轻代占整个堆栈的1/5。Xms=Xmx并且设置了Xmn的情况下，该参数不需要进行设置。
-XX:SurvivorRatio|Eden区与Survivor区的大小比值|-|设置为8，则两个Survivor区与一个Eden区的比值为2:8，一个Survivor区占整个年轻代的1/10
-XX:LargePageSizeInBytes|内存页的大小不可设置过大，会影响Perm的大小|-|=128m
-XX:+UseFastAccessorMethods|原始类型的快速优化|-|-
-XX:+DisableExplicitGC|关闭System.gc()|-|这个参数需要严格的测试
-XX:MaxTenuringThreshold|垃圾最大年龄|-|如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率，该参数只有在串行GC时才有效。
-XX:+AggressiveOpts|加快编译|-|-
-XX:+UseBiasedLocking|锁机制的性能改善|-|-
-Xnoclassgc|禁用垃圾回收|-|-
-XX:SoftRefLRUPolicyMSPerMB|每兆堆空闲空间中SoftReference的存活时间|1s|softly reachable objects will remain alive for some amount of time after the last time they were referenced. The default value is one second of lifetime per free megabyte in the heap
-XX:PretenureSizeThreshold|对象超过多大是直接在旧生代分配|0|单位字节 新生代采用Parallel Scavenge GC时无效，另一种直接在旧生代分配的情况是大的数组对象,且数组中无外部引用对象。
-XX:TLABWasteTargetPercent|TLAB占eden区的百分比|1%|-
-XX:+CollectGen0First|FullGC时是否先YGC|false|-

#### 并行收集器相关参数

参数名称|含义|默认值|备注
---|---|---|---
-XX:+UseParallelGC|Full GC采用parallel MSC(此项待验证)|-|选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下，年轻代使用并发收集，而年老代仍旧使用串行收集。(此项待验证)
-XX:+UseParNewGC|设置年轻代为并行收集|-|可与CMS收集同时使用。JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此值

参考文章：

- 