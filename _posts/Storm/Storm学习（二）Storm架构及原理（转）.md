---
title: "Storm学习（二）Storm架构及原理（转）"
date: 2016-11-07 19:34:53
tags: [Storm]
categories: [Storm]
---

Storm集群环境已经搭建好了，但是在翻译官网的Getting Started的时候感觉有很多概念都不是太理解，所以这篇重点研究一下Storm架构及原理

以下内容转载自：https://my.oschina.net/leejun2005/blog/147607?fromerr=NjSkGlQI

### Storm 与传统的大数据

Storm 与其他大数据解决方案的不同之处在于它的处理方式。Hadoop 在本质上是一个批处理系统。数据被引入 Hadoop 文件系统 (HDFS) 并分发到各个节点进行处理。当处理完成时，结果数据返回到 HDFS 供始发者使用。Storm 支持创建拓扑结构来转换没有终点的数据流。不同于 Hadoop 作业，这些转换从不停止，它们会持续处理到达的数据。

但 Storm 不只是一个传统的大数据分析系统：它是复杂事件处理 (CEP) 系统的一个示例。CEP 系统通常分类为计算和面向检测，其中每个系统都可通过用户定义的算法在 Storm 中实现。举例而言，CEP 可用于识别事件洪流中有意义的事件，然后实时地处理这些事件。

### Storm的基本组件

Storm的集群表面上看和Hadoop的集群非常像。但是在Hadoop上面你运行的是MapReduce的Job, 而在Storm上面你运行的是Topology。它们是非常不一样的 — 一个关键的区别是： 一个MapReduce Job最终会结束， 而一个Topology运永远运行（除非你显式的杀掉他）。

在Storm的集群里面有两种节点： 控制节点(master node)和工作节点(worker node)。控制节点上面运行一个后台程序： Nimbus， 它的作用类似Hadoop里面的JobTracker。Nimbus负责在集群里面分布代码，分配工作给机器， 并且监控状态。

每一个工作节点上面运行一个叫做Supervisor的节点（类似 TaskTracker）。Supervisor会监听分配给它那台机器的工作，根据需要 启动/关闭工作进程。每一个工作进程执行一个Topology（类似 Job）的一个子集；一个运行的Topology由运行在很多机器上的很多工作进程 Worker（类似 Child）组成

##### Storm Topology结构

![Storm Flow](http://img.blog.csdn.net/20161107201042412?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

##### Storm VS MapReduce

![Storm VS MapReduce](http://img.blog.csdn.net/20161107201112288?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Nimbus和Supervisor之间的所有协调工作都是通过一个Zookeeper集群来完成。并且，Nimbus进程和Supervisor都是快速失败（fail-fast)和无状态的。所有的状态要么在Zookeeper里面， 要么在本地磁盘上。这也就意味着你可以用kill -9来杀死Nimbus和Supervisor进程， 然后再重启它们，它们可以继续工作， 就好像什么都没有发生过似的。这个设计使得Storm不可思议的稳定。

### Topologies

为了在Storm上面做实时计算，你要去建立一些Topologies。一个Topology就是一个计算节点所组成的图。Topology里面的每个处理节点都包含处理逻辑，而节点之间的连接则表示数据流动的方向。

运行一个Topology是很简单的。首先，把你所有的代码以及所依赖的jar打进一个jar包。然后运行类似下面的这个命令。

```
storm jar all-your-code.jar backtype.storm.MyTopology arg1 arg2
```

这个命令会运行主类: backtype.strom.MyTopology, 参数是arg1, arg2。这个类的main函数定义这个Topology并且把它提交给Nimbus。storm jar负责连接到Nimbus并且上传jar文件。

因为Topology的定义其实就是一个Thrift结构并且Nimbus就是一个Thrift服务， 有可以用任何语言创建并且提交Topology。上面的方面是用JVM-based语言提交的最简单的方法, 看一下文章: 在生产集群上运行Topology去看看怎么启动以及停止Topologies。

### Stream

Stream是Storm里面的关键抽象。一个Stream是一个没有边界的Tuple序列。Storm提供一些原语来分布式地、可靠地把一个Stream传输进一个新的Stream。比如： 你可以把一个Tweets流传输到热门话题的流。

Storm提供的最基本的处理Stream的原语是Spout和Bolt。你可以实现Spout和Bolt对应的接口以处理你的应用的逻辑。

#### Spout

Spout的流的源头。比如一个Spout可能从Kestrel队列里面读取消息并且把这些消息发射成一个流。又比如一个Spout可以调用Twitter的一个api并且把返回的Tweets发射成一个流。

通常Spout会从外部数据源（队列、数据库等）读取数据，然后封装成Tuple形式，之后发送到Stream中。Spout是一个主动的角色，在接口内部有个nextTuple函数，Storm框架会不停的调用该函数。

![Spout](http://img.blog.csdn.net/20161107202111762?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### Bolt

Bolt可以接收任意多个输入Stream，作一些处理，有些Bolt可能还会发射一些新的Stream。一些复杂的流转换，比如从一些Tweet里面计算出热门话题，需要多个步骤，从而也就需要多个Bolt。Bolt可以做任何事情: 运行函数，过滤Tuple，做一些聚合，做一些合并以及访问数据库等等。

Bolt处理输入的Stream，并产生新的输出Stream。Bolt可以执行过滤、函数操作、Join、操作数据库等任何操作。Bolt是一个被动的角色，其接口中有一个execute(Tuple input)方法，在接收到消息之后会调用此函数，用户可以在此方法中执行自己的处理逻辑。

![Bolt](http://img.blog.csdn.net/20161107202143477?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Spout和Bolt所组成一个网络会被打包成Topology，Topology是Storm里面最高一级的抽象（类Job）， 你可以把Topology提交给Storm的集群来运行。Topology的结构在Topology那一段已经说过了，这里就不再赘述了。

![Topology结构](http://img.blog.csdn.net/20161107202222212?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Topology里面的每一个节点都是并行运行的。在你的Topology里面，你可以指定每个节点的并行度， Storm则会在集群里面分配那么多线程来同时计算。

一个Topology会一直运行直到你显式停止它。Storm自动重新分配一些运行失败的任务，并且Storm保证你不会有数据丢失，即使在一些机器意外停机并且消息被丢掉的情况下。

### 数据模型(Data Model)

Storm使用Tuple来作为它的数据模型。每个Tuple是一堆值，每个值有一个名字，并且每个值可以是任何类型，在我的理解里面一个Tuple可以看作一个没有方法的Java对象。总体来看，Storm支持所有的基本类型、字符串以及字节数组作为Tuple的值类型。你也可以使用你自己定义的类型来作为值类型，只要你实现对应的序列化器(Serializer)。

一个Tuple代表数据流中的一个基本的处理单元，例如一条Cookie日志，它可以包含多个Field，每个Field表示一个属性。

Tuple本来应该是一个Key-Value的Map，由于各个组件间传递的Tuple的字段名称已经事先定义好了，所以Tuple只需要按序填入各个Value，所以就是一个Value List。

一个没有边界的、源源不断的、连续的Tuple序列就组成了Stream。

Topology里面的每个节点必须定义它要发射的Tuple的每个字段。 比如下面这个Bolt定义它所发射的Tuple包含两个字段，类型分别是: double和triple。

```
publicclassDoubleAndTripleBoltimplementsIRichBolt {
    
    privateOutputCollectorBase _collector;
 
    @Override
    publicvoidprepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
        _collector = collector;
    }
 
    @Override
    publicvoidexecute(Tuple input) {
        intval = input.getInteger(0);
        _collector.emit(input,newValues(val*2, val*3));
        _collector.ack(input);
    }
 
    @Override
    publicvoidcleanup() {
    }
 
    @Override
    publicvoiddeclareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(newFields("double","triple"));
    }
}
```
declareOutputFields方法定义要输出的字段 ： ["double", "triple"]。这个Bolt的其它部分我们接下来会解释。

### 一个简单的Topology

让我们来看一个简单的Topology的例子， 我们看一下storm-starter里面的ExclamationTopology:

```
TopologyBuilder builder =newTopologyBuilder();
builder.setSpout(1,newTestWordSpout(),10);
builder.setBolt(2,newExclamationBolt(),3)
        .shuffleGrouping(1);
builder.setBolt(3,newExclamationBolt(),2)
        .shuffleGrouping(2);
```

这个Topology包含一个Spout和两个Bolt。Spout发射单词，每个Bolt在每个单词后面加个”!!!”。这三个节点被排成一条线: Spout发射单词给第一个Bolt，第一个Bolt然后把处理好的单词发射给第二个Bolt。如果Spout发射的单词是["bob"]和["john"], 那么第二个Bolt会发射["bolt!!!!!!"]和["john!!!!!!"]出来。

我们使用setSpout和setBolt来定义Topology里面的节点。这些方法接收我们指定的一个id，一个包含处理逻辑的对象(Spout或者Bolt), 以及你所需要的并行度。

这个包含处理的对象如果是Spout那么要实现IRichSpout的接口，如果是Bolt，那么就要实现IRichBolt接口。

最后一个指定并行度的参数是可选的。它表示集群里面需要多少个Thread来一起执行这个节点。如果你忽略它那么Storm会分配一个线程来执行这个节点。

setBolt方法返回一个InputDeclarer对象，这个对象是用来定义Bolt的输入。这里第一个Bolt声明它要读取Spout所发射的所有的Tuple —— 使用shuffle grouping。而第二个Bolt声明它读取第一个Bolt所发射的Tuple。shuffle grouping表示所有的Tuple会被随机的分发给Bolt的所有Task。给Task分发Tuple的策略有很多种，后面会介绍。

如果你想第二个Bolt读取Spout和第一个Bolt所发射的所有的Tuple， 那么你应该这样定义第二个Bolt:

```
builder.setBolt(3,newExclamationBolt(),5)
            .shuffleGrouping(1)
            .shuffleGrouping(2);
```

让我们深入地看一下这个Topology里面的Spout和Bolt是怎么实现的。Spout负责发射新的Tuple到这个Topology里面来。TestWordSpout从["nathan", "mike", "jackson", "golda", "bertels"]里面随机选择一个单词发射出来。TestWordSpout里面的nextTuple()方法是这样定义的：

```
publicvoidnextTuple() {
    Utils.sleep(100);
    finalString[] words =newString[] {"nathan","mike",
                     "jackson","golda","bertels"};
    finalRandom rand =newRandom();
    finalString word = words[rand.nextInt(words.length)];
    _collector.emit(newValues(word));
}
```

可以看到，实现很简单。

ExclamationBolt把”!!!”拼接到输入tuple后面。我们来看下ExclamationBolt的完整实现。

```
publicstaticclassExclamationBoltimplementsIRichBolt {
    OutputCollector _collector;
 
    publicvoidprepare(Map conf, TopologyContext context,
                        OutputCollector collector) {
        _collector = collector;
    }
 
    publicvoidexecute(Tuple tuple) {
        _collector.emit(tuple,newValues(tuple.getString(0) +"!!!"));
        _collector.ack(tuple);
    }
 
    publicvoidcleanup() {
    }
 
    publicvoiddeclareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(newFields("word"));
    }
}
```

prepare方法提供给Bolt一个Outputcollector用来发射tuple。Bolt可以在任何时候发射Tuple —— 在prepare, execute或者cleanup方法里面, 或者甚至在另一个线程里面异步发射。这里prepare方法只是简单地把OutputCollector作为一个类字段保存下来给后面execute方法使用。

execute方法从Bolt的一个输入接收Tuple(一个Bolt可能有多个输入源)。ExclamationBolt获取Tuple的第一个字段，加上”!!!”之后再发射出去。如果一个Bolt有多个输入源，你可以通过调用Tuple#getSourceComponent方法来知道它是来自哪个输入源的。

execute方法里面还有其它一些事情值得一提：输入Tuple被作为emit方法的第一个参数，并且输入Tuple在最后一行被ack。这些呢都是Storm可靠性API的一部分，后面会解释。

cleanup方法在Bolt被关闭的时候调用，它应该清理所有被打开的资源。但是集群不保证这个方法一定会被执行。比如执行Task的机器down掉了，那么根本就没有办法来调用那个方法。cleanup设计的时候是被用来在local mode的时候才被调用(也就是说在一个进程里面模拟整个storm集群), 并且你想在关闭一些Topology的时候避免资源泄漏。

最后，declareOutputFields定义一个叫做”word”的字段的Tuple。

以local mode运行ExclamationTopology
让我们看看怎么以local mode运行ExclamationToplogy。

Storm的运行有两种模式: 本地模式和分布式模式。在本地模式中，Storm用一个进程里面的线程来模拟所有的Spout和Bolt。本地模式对开发和测试来说比较有用。你运行storm-starter里面的Topology的时候它们就是以本地模式运行的，你可以看到Topology里面的每一个组件在发射什么消息。

在分布式模式下，Storm由一堆机器组成。当你提交Topology给master的时候，你同时也把Topology的代码提交了。master负责分发你的代码并且负责给你的Topolgoy分配工作进程。如果一个工作进程挂掉了， master节点会把认为重新分配到其它节点。关于如何在一个集群上面运行Topology，你可以看看Running topologies on a production cluster文章。

下面是以本地模式运行ExclamationTopology的代码:

```
Config conf =newConfig();
conf.setDebug(true);
conf.setNumWorkers(2);
 
LocalCluster cluster =newLocalCluster();
cluster.submitTopology("test", conf, builder.createTopology());
Utils.sleep(10000);
cluster.killTopology("test");
cluster.shutdown();
```

首先， 这个代码定义通过定义一个LocalCluster对象来定义一个进程内的集群。提交Topology给这个虚拟的集群和提交Topology给分布式集群是一样的。通过调用submitTopology方法来提交Topology， 它接受三个参数：要运行的Topology的名字，一个配置对象以及要运行的Topology本身。

Topology的名字是用来唯一区别一个Topology的，这样你然后可以用这个名字来杀死这个Topology的。前面已经说过了，你必须显式的杀掉一个Topology，否则它会一直运行。

Conf对象可以配置很多东西，下面两个是最常见的：

- TOPOLOGY_WORKERS(setNumWorkers) 定义你希望集群分配多少个工作进程给你来执行这个Topology。Topology里面的每个组件会被需要线程来执行。每个组件到底用多少个线程是通过setBolt和setSpout来指定的。这些线程都运行在工作进程里面。每一个工作进程包含一些节点的一些工作线程。比如，如果你指定300个线程，50个进程，那么每个工作进程里面要执行6个线程，而这6个线程可能属于不同的组件(Spout, Bolt)。你可以通过调整每个组件的并行度以及这些线程所在的进程数量来调整Topology的性能。

- TOPOLOGY_DEBUG(setDebug), 当它被设置成true的话，storm会记录下每个组件所发射的每条消息。这在本地环境调试Topology很有用，但是在线上这么做的话会影响性能的。

感兴趣的话可以去看看Conf对象的Javadoc去看看topology的所有配置。
可以看看创建一个新Storm项目去看看怎么配置开发环境以使你能够以本地模式运行Topology.

运行中的Topology主要由以下三个组件组成的：

- Worker processes（进程）
- Executors (threads)（线程）
- Tasks

![Worker](http://img.blog.csdn.net/20161107205213355?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Spout或者Bolt的Task个数一旦指定之后就不能改变了，而Executor的数量可以根据情况来进行动态的调整。默认情况下# executor = #tasks即一个Executor中运行着一个Task

![Executor](http://img.blog.csdn.net/20161107205238449?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![Topology](http://img.blog.csdn.net/20161107205311189?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![conf](http://img.blog.csdn.net/20161107205348799?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 流分组策略(Stream Grouping)

流分组策略告诉Topology如何在两个组件之间发送Tuple。要记住，Spouts和Bolts以很多Task的形式在Topology里面同步执行。如果从Task的粒度来看一个运行的Topology，它应该是这样的:

![Grouping](http://img.blog.csdn.net/20161107210310422?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
 
从Task角度来看Topology

当Bolt A的一个task要发送一个Tuple给Bolt B， 它应该发送给Bolt B的哪个Task呢？

Stream Grouping专门回答这种问题的。在我们深入研究不同的Stream Grouping之前，让我们看一下storm-starter里面的另外一个Topology。WordCountTopology读取一些句子，输出句子里面每个单词出现的次数.

```
TopologyBuilder builder =newTopologyBuilder();
 
builder.setSpout(1,newRandomSentenceSpout(),5);
builder.setBolt(2,newSplitSentence(),8)
        .shuffleGrouping(1);
builder.setBolt(3,newWordCount(),12)
        .fieldsGrouping(2,newFields("word"));
```

SplitSentence对于句子里面的每个单词发射一个新的Tuple, WordCount在内存里面维护一个单词->次数的Mapping，WordCount每收到一个单词，它就更新内存里面的统计状态。

有好几种不同的Stream Grouping:

最简单的Grouping是shuffle grouping, 它随机发给任何一个Task。上面例子里面RandomSentenceSpout和SplitSentence之间用的就是shuffle grouping, shuffle grouping对各个Task的Tuple分配的比较均匀。

一种更有趣的Grouping是fields grouping, SplitSentence和WordCount之间使用的就是fields grouping, 这种Grouping机制保证相同Field值的Tuple会去同一个Task，这对于WordCount来说非常关键，如果同一个单词不去同一个task，那么统计出来的单词次数就不对了。

fields grouping是Stream合并，Stream聚合以及很多其它场景的基础。在背后呢，fields grouping使用的一致性哈希来分配Tuple的。

还有一些其它类型的Stream Grouping. 你可以在Concepts一章里更详细的了解。

下面是一些常用的 “路由选择” 机制：

Storm的Grouping即消息的Partition机制。当一个Tuple被发送时，如何确定将它发送个某个（些）Task来处理？？

- ShuffleGrouping：随机选择一个Task来发送。
- FieldGrouping：根据Tuple中Fields来做一致性hash，相同hash值的Tuple被发送到相同的Task。
- AllGrouping：广播发送，将每一个Tuple发送到所有的Task。
- GlobalGrouping：所有的Tuple会被发送到某个Bolt中的id最小的那个Task。
- NoneGrouping：不关心Tuple发送给哪个Task来处理，等价于ShuffleGrouping。
- DirectGrouping：直接将Tuple发送到指定的Task来处理。

### 使用别的语言来定义Bolt

Bolt可以使用任何语言来定义。用其它语言定义的Bolt会被当作子进程(subprocess)来执行， Storm使用JSON消息通过stdin/stdout来和这些subprocess通信。这个通信协议是一个只有100行的库，Storm团队给这些库开发了对应的Ruby, Python和Fancy版本。

下面是WordCountTopology里面的SplitSentence的定义:

```
publicstaticclassSplitSentenceextendsShellBoltimplementsIRichBolt {
    publicSplitSentence() {
        super("python","splitsentence.py");
    }
 
    publicvoiddeclareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(newFields("word"));
    }
}
```

SplitSentence继承自ShellBolt并且声明这个Bolt用Python来运行，并且参数是: splitsentence.py。下面是splitsentence.py的定义:

```
importstorm
 
classSplitSentenceBolt(storm.BasicBolt):
    defprocess(self, tup):
        words=tup.values[0].split(" ")
        forwordinwords:
          storm.emit([word])
 
SplitSentenceBolt().run()
```

更多有关用其它语言定义Spout和Bolt的信息， 以及用其它语言来创建topology的 信息可以参见: Using non-JVM languages with Storm.

### 可靠的消息处理

在这个教程的前面，我们跳过了有关Tuple的一些特征。这些特征就是Storm的可靠性API： Storm如何保证Spout发出的每一个Tuple都被完整处理。看看《storm如何保证消息不丢失》以更深入了解storm的可靠性API.

Storm允许用户在Spout中发射一个新的源Tuple时为其指定一个MessageId，这个MessageId可以是任意的Object对象。多个源Tuple可以共用同一个MessageId，表示这多个源Tuple对用户来说是同一个消息单元。Storm的可靠性是指Storm会告知用户每一个消息单元是否在一个指定的时间内被完全处理。完全处理的意思是该MessageId绑定的源Tuple以及由该源Tuple衍生的所有Tuple都经过了Topology中每一个应该到达的Bolt的处理。

![Message](http://img.blog.csdn.net/20161107210353469?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

在Spout中由message 1绑定的tuple1和tuple2分别经过bolt1和bolt2的处理，然后生成了两个新的Tuple，并最终流向了bolt3。当bolt3处理完之后，称message 1被完全处理了。

Storm中的每一个Topology中都包含有一个Acker组件。Acker组件的任务就是跟踪从Spout中流出的每一个messageId所绑定的Tuple树中的所有Tuple的处理情况。如果在用户设置的最大超时时间内这些Tuple没有被完全处理，那么Acker会告诉Spout该消息处理失败，相反则会告知Spout该消息处理成功。

那么Acker是如何记录Tuple的处理结果呢？？

A xor A = 0.

A xor B…xor B xor A = 0，其中每一个操作数出现且仅出现两次。

在Spout中，Storm系统会为用户指定的MessageId生成一个对应的64位的整数，作为整个Tuple Tree的RootId。RootId会被传递给Acker以及后续的Bolt来作为该消息单元的唯一标识。同时，无论Spout还是Bolt每次新生成一个Tuple时，都会赋予该Tuple一个唯一的64位整数的Id。

当Spout发射完某个MessageId对应的源Tuple之后，它会告诉Acker自己发射的RootId以及生成的那些源Tuple的Id。而当Bolt处理完一个输入Tuple并产生出新的Tuple时，也会告知Acker自己处理的输入Tuple的Id以及新生成的那些Tuple的Id。Acker只需要对这些Id进行异或运算，就能判断出该RootId对应的消息单元是否成功处理完成了。

参考文章：

- https://my.oschina.net/leejun2005/blog/147607?fromerr=NjSkGlQI
- https://www.ibm.com/developerworks/cn/opensource/os-twitterstorm/#list1