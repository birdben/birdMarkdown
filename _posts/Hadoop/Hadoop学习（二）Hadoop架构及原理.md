---
title: "Hadoop学习（二）Hadoop架构及原理"
date: 2016-09-12 10:29:27
tags: [Hadoop原理架构体系, HDFS]
categories: [Hadoop]
---

通过上一章Hadoop完全分布式集群的环境搭建，遇到了各种各样的疑问，包括MRv1代和MRv2代的区别，对很多Hadoop进程的意义都进行了详细的了解，包括一些Hadoop组件的原理

先来说说我的疑问

- 有NameNode和DataNode，有ResourceManager和NodeManager，为什么没有网上说的JobTracker和TaskTracker
- SecondaryNameNode是做什么的
- MRv1和MRv2有什么区别
- YARN是做什么的
- core-site.xml, hdfs-site.xml, mapred-site.xml, yarn-site.xml这几个配置文件是配置什么的
- HDFS默认的端口号是8020还是9000

看了上面的疑问，我想如果对Hadoop有研究的人应该都知道答案了。但是对于我这样的初学者理解起来还是费了点时间，下面我们将逐渐解答上面的疑团。

### MRv1和MRv2的区别

先说说MRv1和MRv2，MRv1就是MapReduce v1版本和MapReduce v2版本，从 Hadoop 0.23.0 版本开始，Hadoop 的 MapReduce 框架完全重构，发生了根本的变化。新的 Hadoop MapReduce 框架命名为 MapReduceV2 或者叫 Yarn。这里我们使用的Hadoop-2.7.1版本，也就是用的YARN版本。

#### YARN的好处

与MRv1相比，YARN不再是一个单纯的计算框架，而是一个框架管理器，用户可以将各种各样的计算框架移植到YARN之上，由YARN进行统一管理和资源分配，由于将现有框架移植到YARN之上需要一定的工作量，当前YARN仅可运行MapReduce这种离线计算框架。

我们知道，不存在一种统一的计算框架适合所有应用场景，也就是说，如果你想设计一种计算框架，可以高效地进行离线计算、在线计算、流式计算、内存计算等，是不可能的。既然没有全能的计算框架，为什么不开发一个容纳和管理各种计算框架的框架管理平台（实际上是资源管理平台）呢，而YARN正是干这件事情的东西。

YANR本质上是一个资源统一管理系统，这一点与几年前的mesos（http://www.mesosproject.org/），更早的Torque（http://www.adaptivecomputing.com/products/open-source/torque/）基本一致。将各种框架运行在YARN之上，可以实现框架的资源统一管理和分配，使他们共享一个集群，而不是“一个框架一个集群”，这可大大降低运维成本和硬件成本。

下面列举了比较流行的多计算框架

- MapReduce:  这个框架人人皆知，它是一种离线计算框架，将一个算法抽象成Map和Reduce两个阶段进行处理，非常适合数据密集型计算。
- Spark:  我们知道，MapReduce计算框架不适合（不是不能做，是不适合，效率太低）迭代计算（常见于machine learning领域，比如PageRank）和交互式计算（data mining领域，比如SQL查询），MapReduce是一种磁盘计算框架，而Spark则是一种内存计算框架，它将数据尽可能放到内存中以提高迭代应用和交互式应用的计算效率。官方首页：http://spark-project.org/
- Storm:  MapReduce也不适合进行流式计算、实时分析，比如广告点击计算等，而Storm则更擅长这种计算、它在实时性要远远好于MapReduce计算框架。官方首页：http://storm-project.net/

在YARN中，各种计算框架不再是作为一个服务部署到集群的各个节点上（比如MapReduce框架，不再需要部署JobTracler、TaskTracker等服务），而是被封装成一个用户程序库（lib）存放在客户端，当需要对计算框架进行升级时，只需升级用户程序库即可，多么容易！

再来看看MRv1和MRv2的对比。

#### MRv1结构图

- NameNode : HDFS分发节点
- DataNode : HDFS数据节点
- JobTracker : 首先用户程序 (JobClient) 提交了一个 job，job 的信息会发送到 Job Tracker 中，Job Tracker 是 Map-reduce 框架的中心，他需要与集群中的机器定时通信 (heartbeat), 需要管理哪些程序应该跑在哪些机器上，需要管理所有 job 失败、重启等操作。
- TaskTracker : TaskTracker 是 Map-reduce 集群中每台机器都有的一个部分，他做的事情主要是监视自己所在机器的资源情况。TaskTracker 同时监视当前机器的 tasks 运行状况。TaskTracker 需要把这些信息通过 heartbeat 发送给 JobTracker，JobTracker 会搜集这些信息以给新提交的 job 分配运行在哪些机器上。

![MRv1](http://img.blog.csdn.net/20160924114720210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

当一个客户端向一个 Hadoop 集群发出一个请求时，此请求由 JobTracker 管理。JobTracker 与 NameNode 联合将工作分发到离它所处理的数据尽可能近的位置。NameNode 是文件系统的主系统，提供元数据服务来执行数据分发和复制。JobTracker 将 Map 和 Reduce 任务安排到一个或多个 TaskTracker 上的可用插槽中。TaskTracker 与 DataNode（分布式文件系统）一起对来自 DataNode 的数据执行 Map 和 Reduce 任务。当 Map 和 Reduce 任务完成时，TaskTracker 会告知 JobTracker，后者确定所有任务何时完成并最终告知客户作业已完成。

从 图 1 中可以看到，MRv1 实现了一个相对简单的集群管理器来执行 MapReduce 处理。MRv1 提供了一种分层的集群管理模式，其中大数据作业以单个 Map 和 Reduce 任务的形式渗入一个集群，并最后聚合成作业来报告给用户。但这种简单性有一些隐秘，不过也不是很隐秘的问题。 

##### MRv1 的缺陷

- JobTracker 是 Map-reduce 的集中处理点，存在单点故障。
- JobTracker 完成了太多的任务，造成了过多的资源消耗，当 map-reduce job 非常多的时候，会造成很大的内存开销，潜在来说，也增加了 JobTracker fail 的风险，这也是业界普遍总结出老 Hadoop 的 Map-Reduce 只能支持 4000 节点主机的上限。
- 在 TaskTracker 端，以 map/reduce task 的数目作为资源的表示过于简单，没有考虑到 cpu/ 内存的占用情况，如果两个大内存消耗的 task 被调度到了一块，很容易出现 OOM。
- 在 TaskTracker 端，把资源强制划分为 map task slot 和 reduce task slot, 如果当系统中只有 map task 或者只有 reduce task 的时候，会造成资源的浪费，也就是前面提过的集群资源利用的问题。
- 源代码层面分析的时候，会发现代码非常的难读，常常因为一个 class 做了太多的事情，代码量达 3000 多行，，造成 class 的任务不清晰，增加 bug 修复和版本维护的难度。
- 从操作的角度来看，现在的 Hadoop MapReduce 框架在有任何重要的或者不重要的变化 ( 例如 bug 修复，性能提升和特性化 ) 时，都会强制进行系统级别的升级更新。更糟的是，它不管用户的喜好，强制让分布式集群系统的每一个用户端同时更新。这些更新会让用户为了验证他们之前的应用程序是不是适用新的 Hadoop 版本而浪费大量时间。

#### MRv2结构图

- NameNode : HDFS分发节点
- DataNode : HDFS数据节点
- ResourceManager : MR资源管理
- NodeManager : NodeManager是执行应用程序的容器，监控应用程序的资源使用情况 (CPU，内存，硬盘，网络 ) 并且向调度器汇报。
- ApplicationMaster : ApplicationMaster是向调度器索要适当的资源容器，运行任务，跟踪应用程序的状态和监控它们的进程，处理任务的失败原因。 
- SecondaryNameNode

![MRv2](http://img.blog.csdn.net/20160924114749368?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

重构根本的思想是将 JobTracker 两个主要的功能分离成单独的组件，这两个功能是资源管理和任务调度 / 监控。新的资源管理器全局管理所有应用程序计算资源的分配，每一个应用的 ApplicationMaster 负责相应的调度和协调。一个应用程序无非是一个单独的传统的 MapReduce 任务或者是一个 DAG( 有向无环图 ) 任务。ResourceManager 和每一台机器的节点管理服务器能够管理用户在那台机器上的进程并能对计算进行组织。

事实上，每一个应用的 ApplicationMaster 是一个详细的框架库，它结合从 ResourceManager 获得的资源和 NodeManager 协同工作来运行和监控任务。

上图中 ResourceManager 支持分层级的应用队列，这些队列享有集群一定比例的资源。从某种意义上讲它就是一个纯粹的调度器，它在执行过程中不对应用进行监控和状态跟踪。同样，它也不能重启因应用失败或者硬件错误而运行失败的任务。

ResourceManager 是基于应用程序对资源的需求进行调度的 ; 每一个应用程序需要不同类型的资源因此就需要不同的容器。资源包括：内存，CPU，磁盘，网络等等。可以看出，这同现 Mapreduce 固定类型的资源使用模型有显著区别，它给集群的使用带来负面的影响。资源管理器提供一个调度策略的插件，它负责将集群资源分配给多个队列和应用程序。调度插件可以基于现有的能力调度和公平调度模型。

上图中 NodeManager 是每一台机器框架的代理，是执行应用程序的容器，监控应用程序的资源使用情况 (CPU，内存，硬盘，网络 ) 并且向调度器汇报。

每一个应用的 ApplicationMaster 的职责有：向调度器索要适当的资源容器，运行任务，跟踪应用程序的状态和监控它们的进程，处理任务的失败原因。 

#### 新旧 Hadoop MapReduce 框架比对
让我们来对新旧 MapReduce 框架做详细的分析和对比，可以看到有以下几点显著变化：
首先客户端不变，其调用 API 及接口大部分保持兼容，这也是为了对开发使用者透明化，使其不必对原有代码做大的改变 ( 详见 2.3 Demo 代码开发及详解)，但是原框架中核心的 JobTracker 和 TaskTracker 不见了，取而代之的是 ResourceManager, ApplicationMaster 与 NodeManager 三个部分。

我们来详细解释这三个部分，首先 ResourceManager 是一个中心的服务，它做的事情是调度、启动每一个 Job 所属的 ApplicationMaster、另外监控 ApplicationMaster 的存在情况。细心的读者会发现：Job 里面所在的 task 的监控、重启等等内容不见了。这就是 AppMst 存在的原因。ResourceManager 负责作业与资源的调度。接收 JobSubmitter 提交的作业，按照作业的上下文 (Context) 信息，以及从 NodeManager 收集来的状态信息，启动调度过程，分配一个 Container 作为 App Mstr
NodeManager 功能比较专一，就是负责 Container 状态的维护，并向 RM 保持心跳。

ApplicationMaster 负责一个 Job 生命周期内的所有工作，类似老的框架中 JobTracker。但注意每一个 Job（不是每一种）都有一个 ApplicationMaster，它可以运行在 ResourceManager 以外的机器上。

Yarn 框架相对于老的 MapReduce 框架什么优势呢？我们可以看到：
这个设计大大减小了 JobTracker（也就是现在的 ResourceManager）的资源消耗，并且让监测每一个 Job 子任务 (tasks) 状态的程序分布式化了，更安全、更优美。

在新的 Yarn 中，ApplicationMaster 是一个可变更的部分，用户可以对不同的编程模型写自己的 AppMst，让更多类型的编程模型能够跑在 Hadoop 集群中，可以参考 hadoop Yarn 官方配置模板中的 mapred-site.xml 配置。

对于资源的表示以内存为单位 ( 在目前版本的 Yarn 中，没有考虑 cpu 的占用 )，比之前以剩余 slot 数目更合理。

老的框架中，JobTracker 一个很大的负担就是监控 job 下的 tasks 的运行状况，现在，这个部分就扔给 ApplicationMaster 做了，而 ResourceManager 中有一个模块叫做 ApplicationsMasters( 注意不是 ApplicationMaster)，它是监测 ApplicationMaster 的运行状况，如果出问题，会将其在其他机器上重启。

Container 是 Yarn 为了将来作资源隔离而提出的一个框架。这一点应该借鉴了 Mesos 的工作，目前是一个框架，仅仅提供 java 虚拟机内存的隔离 ,hadoop 团队的设计思路应该后续能支持更多的资源调度和控制 , 既然资源表示成内存量，那就没有了之前的 map slot/reduce slot 分开造成集群资源闲置的尴尬情况。

引入YARN作为通用资源调度平台后，Hadoop得以支持多种计算框架，如MapReduce、Spark、Storm等。MRv1是Hadoop1中的MapReduce，MRv2是Hadoop2中的MapReduce。下面是MRv1和MRv2之间的一些基本变化：

MRv1包括三个部分：运行时环境（jobtracker和tasktracker）、编程模型（MapReduce）、数据处理引擎（Map任务和Reduce任务）
MRv2中，重用了MRv1中的编程模型和数据处理引擎。但是运行时环境被重构了。jobtracker被拆分成了通用的资源调度平台YARN和负责各个计算框架的任务调度模型AM。
MRv1中任务是运行在Map slot和Reduce slot中的，计算节点上的Map slot资源和Reduce slot资源不能重用。而MRv2中任务是运行在container中的，map任务结束后，相应container结束，空闲出来的资源可以让reduce使用。

参考文章：

- http://dongxicheng.org/mapreduce-nextgen/what-can-we-benifit-from-yarn/


### Hadoop的默认端口
Hadoop集群的各部分一般都会使用到多个端口，有些是daemon之间进行交互之用，有些是用于RPC访问以及HTTP访问。而随着Hadoop周边组件的增多，完全记不住哪个端口对应哪个应用，特收集记录如此，以便查询。

这里包含我们使用到的组件：HDFS, YARN, HBase, Hive, ZooKeeper。

组件		|Daemon			|端口		|配置	|说明
----		|----				|----		|----	|-----
HDFS		|DataNode			|50010		|dfs.datanode.address	|datanode服务端口，用于数据传输
HDFS		|DataNode			|50075		|dfs.datanode.http.address	|http服务的端口
HDFS		|DataNode			|50475		|dfs.datanode.https.address	|https服务的端口
HDFS		|DataNode			|50020		|dfs.datanode.ipc.address	|ipc服务的端口
HDFS		|NameNode			|50070		|dfs.namenode.http-address	|http服务的端口
HDFS		|NameNode			|50470		|dfs.namenode.https-address	|https服务的端口
HDFS		|NameNode			|8020		|fs.defaultFS	|接收Client连接的RPC端口，用于获取文件系统metadata信息。
HDFS		|journalnode		|8485		|dfs.journalnode.rpc-address	|RPC服务
HDFS		| journalnode		|8480		|dfs.journalnode.http-address	HTTP服务
HDFS		|ZKFC				|8019		|dfs.ha.zkfc.port	|ZooKeeper FailoverController，用于NN HA
YARN		|ResourceManager	|8032		|yarn.resourcemanager.address	|RM的applications manager(ASM)端口
YARN		|ResourceManager	|8030		|yarn.resourcemanager.scheduler.address	|scheduler组件的IPC端口
YARN		|ResourceManager	|8031		|yarn.resourcemanager.resource-tracker.address	|IPC
YARN		|ResourceManager	|8033		|yarn.resourcemanager.admin.address	|IPC
YARN		|ResourceManager	|8088		|yarn.resourcemanager.webapp.address	|http服务端口
YARN		|NodeManager		|8040		|yarn.nodemanager.localizer.address	|localizer IPC
YARN		|NodeManager		|8042		|yarn.nodemanager.webapp.address	|http服务端口
YARN		|NodeManager		|8041		|yarn.nodemanager.address	|NM中container manager的端口
YARN		|JobHistory Server	|10020	|mapreduce.jobhistory.address	|IPC
YARN		|JobHistory Server	|19888	|mapreduce.jobhistory.webapp.address	|http服务端口
HBase		|Master			|60000		|hbase.master.port	|IPC
HBase		|Master			|60010		|hbase.master.info.port	|http服务端口
HBase		|RegionServer		|60020		|hbase.regionserver.port	|IPC
HBase		|RegionServer		|60030		|hbase.regionserver.info.port	|http服务端口
HBase		|HQuorumPeer		|2181		|hbase.zookeeper.property.clientPort	|HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase		|HQuorumPeer		|2888		|hbase.zookeeper.peerport	|HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
HBase		|HQuorumPeer		|3888		|hbase.zookeeper.leaderport	|HBase-managed ZK mode，使用独立的ZooKeeper集群则不会启用该端口。
Hive		|Metastore		|9083		|/etc/default/hive-metastore中export PORT=<port>来更新默认端口
Hive		|HiveServer		|10000		|/etc/hive/conf/hive-env.sh中export HIVE\_SERVER2\_THRIFT\_PORT=<port>来更新默认端口
ZooKeeper	|Server			|2181		|/etc/zookeeper/conf/zoo.cfg中clientPort=<port>	|对客户端提供服务的端口
ZooKeeper	|Server			|2888		|/etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分	|follower用来连接到leader，只在leader上监听该端口。
ZooKeeper	|Server			|3888		|/etc/zookeeper/conf/zoo.cfg中server.x=[hostname]:nnnnn[:nnnnn]，标蓝部分	|用于leader选举的。只在electionAlg是1,2或3(默认)时需要。

参考文章：

- http://blog.csdn.net/zzhongcy/article/details/19912577
- http://blog.csdn.net/wulantian/article/details/46341043

### Hadoop的IPC机制

参考文章：

- http://www.cnblogs.com/xuxm2007/archive/2012/06/22/2558599.html
- https://my.oschina.net/zavakid/blog/119020
- http://seandeng888.iteye.com/blog/2160714

### YARN在Hadoop中的作用

### YARN的ResourceManager

### SecondaryNameNode节点

参考文章：

- http://www.cnblogs.com/likehua/p/4023777.html

### MapReduce shuffle过程原理

### MapReduce原理分析

最简单的MapReduce应用程序至少包含 3 个部分：一个 Map 函数、一个 Reduce 函数和一个 main 函数。在运行一个mapreduce计算任务时候，任务过程被分为两个阶段：map阶段和reduce阶段，每个阶段都是用键值对（key/value）作为输入（input）和输出（output）。main 函数将作业控制和文件输入/输出结合起来。

所以我们编写了下面的三个Java类

- WordCount：MapReduce程序入口类
- TokenizerMapper：Map并行读取文本，对读取的单词进行map操作，每个词都以<key,value>形式生成。
- IntSumReducer：Reduce操作是对map的结果进行排序，合并，最后得出词频。

简单来说，map负责把任务分解成多个任务，reduce负责把分解后多任务处理的结果汇总起来。

#### MapReduce的执行过程

![MapReduce执行过程](http://img.blog.csdn.net/20161030164443369?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

MapReduce的执行过程主要包含是三个阶段：Map阶段、Shuffle阶段、Reduce阶段

- Map阶段：
	* 分片（Split）：map阶段的输入通常是HDFS上文件，在运行Mapper前，FileInputFormat会将输入文件分割成多个split ——1个split至少包含1个HDFS的Block（默认为64M）；然后每一个分片运行一个map进行处理。
	* 执行（Map）：对输入分片中的每个键值对调用map()函数进行运算，然后输出一个结果键值对。

		Partitioner：对map()的输出进行partition，即根据key或value及reduce的数量来决定当前的这对键值对最终应该交由哪个reduce处理。默认是对key哈希后再以reduce task数量取模，默认的取模方式只是为了避免数据倾斜。然后该key/value对以及partitionIdx的结果都会被写入环形缓冲区。
	
	* 溢写（Spill）：map输出写在内存中的环形缓冲区，默认当缓冲区满80%，启动溢写线程，将缓冲的数据写出到磁盘。

		Sort：在溢写到磁盘之前，使用快排对缓冲区数据按照partitionIdx, key排序。（每个partitionIdx表示一个分区，一个分区对应一个reduce）

		Combiner：如果设置了Combiner，那么在Sort之后，还会对具有相同key的键值对进行合并，减少溢写到磁盘的数据量。
	
	* 合并（Merge）：溢写可能会生成多个文件，这时需要将多个文件合并成一个文件。合并的过程中会不断地进行 sort & combine 操作，最后合并成了一个已分区且已排序的文件。

- Shuffle阶段：广义上Shuffle阶段横跨Map端和Reduce端，在Map端包括Spill过程，在Reduce端包括copy和merge/sort过程。通常认为Shuffle阶段就是将map的输出作为reduce的输入的过程
	* Copy过程：Reduce端启动一些copy线程，通过HTTP方式将map端输出文件中属于自己的部分拉取到本地。Reduce会从多个map端拉取数据，并且每个map的数据都是有序的。
	* Merge过程：Copy过来的数据会先放入内存缓冲区中，这里的缓冲区比较大；当缓冲区数据量达到一定阈值时，将数据溢写到磁盘（与map端类似，溢写过程会执行 sort & combine）。如果生成了多个溢写文件，它们会被merge成一个有序的最终文件。这个过程也会不停地执行 sort & combine 操作。

- Reduce阶段：Shuffle阶段最终生成了一个有序的文件作为Reduce的输入，对于该文件中的每一个键值对调用reduce()方法，并将结果写到HDFS。

参考文章：

- http://langyu.iteye.com/blog/992916
- http://blog.itpub.net/29754888/viewspace-1704959/