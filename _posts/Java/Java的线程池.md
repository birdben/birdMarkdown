---
title: "Java的线程池"
date: 2017-08-09 15:09:25
tags: [Java]
categories: [Java]
---

### 1. Java线程池（java.util.concurrent.ThreadPoolExecutor）

大多数JVM6上的应用采用的线程池都是JDK自带的线程池，之所以把成熟的Java线程池进行罗嗦说明，是因为该线程池的行为与我们想象的有点出入。Java线程池有几个重要的配置参数：

- corePoolSize：核心线程数（最新线程数）
- maximumPoolSize：最大线程数，超过这个数量的任务会被拒绝，用户可以通过RejectedExecutionHandler接口自定义处理方式
- keepAliveTime：线程保持活动的时间
- workQueue：工作队列，存放执行的任务

Java线程池需要传入一个Queue参数（workQueue）用来存放执行的任务，而对Queue的不同选择，线程池有完全不同的行为：

- SynchronousQueue： 一个无容量的等待队列，一个线程的insert操作必须等待另一线程的remove操作，采用这个Queue线程池将会为每个任务分配一个新线程
- LinkedBlockingQueue ： 无界队列，采用该Queue，线程池将忽略 maximumPoolSize参数，仅用corePoolSize的线程处理所有的任务，未处理的任务便在LinkedBlockingQueue中排队
- ArrayBlockingQueue： 有界队列，在有界队列和 maximumPoolSize的作用下，程序将很难被调优：更大的Queue和小的maximumPoolSize将导致CPU的低负载；小的Queue和大的池，Queue就没起动应有的作用。

其实我们的要求很简单，希望线程池能跟连接池一样，能设置最小线程数、最大线程数，当最小数 < 任务 < 最大数时，应该分配新的线程处理；当任务 > 最大数时，应该等待有空闲线程再处理该任务。

但线程池的设计思路是，任务应该放到Queue中，当Queue放不下时再考虑用新线程处理，如果Queue满且无法派生新线程，就拒绝该任务。设计导致 “先放等执行”、“放不下再执行”、“拒绝不等待”。所以，根据不同的Queue参数，要提高吞吐量不能一味地增大maximumPoolSize。

当然，要达到我们的目标，必须对线程池进行一定的封装，幸运的是ThreadPoolExecutor中留了足够的自定义接口以帮助我们达到目标。我们封装的方式是：

以SynchronousQueue作为参数，使maximumPoolSize发挥作用，以防止线程被无限制的分配，同时可以通过提高maximumPoolSize来提高系统吞吐量

自定义一个RejectedExecutionHandler，当线程数超过maximumPoolSize时进行处理，处理方式为隔一段时间检查线程池是否可以执行新Task，如果可以把拒绝的Task重新放入到线程池，检查的时间依赖keepAliveTime的大小。

### 2. 连接池（org.apache.commons.dbcp.BasicDataSource）

在使用org.apache.commons.dbcp.BasicDataSource的时候，因为之前采用了默认配置，所以当访问量大时，通过JMX观察到很多Tomcat线程都阻塞在BasicDataSource使用的Apache ObjectPool的锁上，直接原因当时是因为BasicDataSource连接池的最大连接数设置的太小，默认的BasicDataSource配 置，仅使用8个最大连接。

我还观察到一个问题，当较长的时间不访问系统，比如2天，DB上的Mysql会断掉所有的连接，导致连接池中缓存的连接不能用。为了解决这些问题，我们充分研究了BasicDataSource，发现了一些优化的点：

Mysql默认支持100个链接，所以每个连接池的配置要根据集群中的机器数进行，如有2台服务器，可每个设置为60

- initialSize：参数是一直打开的连接数
- minEvictableIdleTimeMillis：该参数设置每个连接的空闲时间，超过这个时间连接将被关闭
- timeBetweenEvictionRunsMillis：后台线程的运行周期，用来检测过期连接
- maxActive：最大能分配的连接数
- maxIdle：最大空闲数，当连接使用完毕后发现连接数大于maxIdle，连接将被直接关闭。只有initialSize < x < maxIdle的连接将被定期检测是否超期。这个参数主要用来在峰值访问时提高吞吐量。
- initialSize是如何保持的？经过研究代码发现，BasicDataSource会关闭所有超期的连接，然后再打开initialSize数量的连接，这个特性与minEvictableIdleTimeMillis、 timeBetweenEvictionRunsMillis一起保证了所有超期的initialSize连接都会被重新连接，从而避免了Mysql长时间无动作会断掉连接的问题。


