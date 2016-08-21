---
title: "Akka工作原理"
date: 2015-11-12 11:50:53
tags: [Akka]
categories: [Akka]
---

![Akka](https://raw.githubusercontent.com/arunma/blogimages/master/Akka1/TeacherRequestFlowSimulatedApp.png)

### Akka中的角色
- ProducerActor(StudentActor)
- ConsumerActor(TeacherActor，onReceive方法接收消息)
- ActorRef(tell方法，发送消息给MessageDispatcher消息派发器)
- ActorSystem(actorOf方法，创建ActorRef，ActorRef就是ConsumerActor的Proxy)
- MailBox
- Dispatcher
- Message

### Akka工作流程
1. ProducerActor(StudentActor)创建了一个叫ActorSystem的东西。
2. 他通过ActorSystem来创建了一个叫ActorRef的对象。QuoteRequest消息就是发送给ActorRef的（它是TeacherActor的一个代理）
3. ActorRef将消息发送给Dispatcher
4. Dispatcher将消息投递到目标Actor的邮箱中。
5. 随后Dispatcher将Mailbox扔给一个线程去执行（这点下节会重点讲到）
6. MailBox将消息出队并最终将其委托给真实的Teacher Actor的接收方法去处理。

#### 创建ActorSystem

ActorSystem是进入到Actor的世界的一扇大门。通过它你可以创建或中止Actor。甚至还可以把整个Actor环境给关闭掉。

另一方面来说，Actor是一个分层的结构，ActorSystem之于Actor有点类似于java.lang.Object或者scala.Any的角色——也就是说，它是所有Actor的根对象。当你通过ActorSystem的actorOf方法创建了一个Actor时，你其实创建的是ActorSystem下面的一个Actor。

![ActorSystem](https://raw.githubusercontent.com/arunma/blogimages/master/Akka1/ActorSystemActorCreation.png)

#### 创建ActorRef(ConsumerActor（TeacherActor）的Proxy)

ActorRef myActor = system.actorOf(Props.create(MyUntypedActor.class), "myactor");

actorOf是ActorSystem中创建Actor的方法。但是正如你所看到的，它并不会返回我们所需要的TeacherActor。它返回的是一个ActorRef。

这个ActorRef扮演了真实的Actor的一个代理的角色。客户端并不会直接和Actor通信。这也正是Actor模型中避免直接访问TeacherActor中任何的自定义/私有方法或者变量的一种方式。

再重复一遍，消息只会发送给ActorRef，最终才会到达真正的Actor。你是绝对无法直接和Actor进行通信的。如果你真的找到了什么拙劣的方式来直接通信，大家会恨你入骨的。

#### 将消息发送给代理

还是只有一行代码。你只需告诉说把QuoteRequest消息发送到ActorRef就好了。Actor中的这个告诉的方式就是一个!号。（ActorRef中确实也有一个tell方法，不过它只是把这个调用委托给了！号）

#### 分发器及邮箱

ActorRef把消息处理功能委托给了Dispatcher。实际上，当我们创建ActorSystem和ActorRef的时候，就已经创建了一个Dispatcher和MailBox了。我们来看下它们是干什么的。

#### 邮箱

每个Actor都有一个MailBox（后面会介绍一种特殊的情况）。在我们这个比喻当中，每个老师也有一个邮箱。老师得去检查邮箱并处理消息。在Actor的世界中，则是另一种形式——邮箱一有机会就会要求Actor去完成自己的任务。

同样的，邮箱里也有一个队列来以FIFO的方式来存储并处理消息——它和实际的邮箱还有点不同，真实的邮箱新的信总是在最上面的。

#### 分发器

Dispatcher会完成一些很酷的事。从它的角度来看，它只是从ActorRef中取出一条消息然后将它传给了MailBox。但是，在这后面发生了一件不可意义的事情：

Dispatcher会封装一个ExecutorService(ForkJoinPoll或者ThreadPoolExecutor)。它把MailBox扔到ExecutorService中去运行。

是的。我们看到MailBox中包含了队列里面的消息。由于Executor得去执行MailBox，所以它得是一个Thread类型。是的没错。MailBox的声明及构造器就是这样的。

#### ConsumerActor(TeacherActor)

当MailBox的run方法运行的时候，它会从队列中取出一条消息，然后将它传给Actor去处理。

当你把消息传给ActorRef的时候，最终调用的实际是目标Actor里面的一个receive方法。

TeacherActor只是一个很简单的类，它有一个名言的列表，而receive方法很明显就是用来处理消息的。

TeacherActor的receive方法的模式匹配只会匹配一种消息——QuoteRequest （事实上，模式匹配中最好匹配下默认的情况，不过这个就说来话长了）

receive方法做的就是

1. 匹配QuoteRequest的模式
2. 从名言列表中随机选取一条
3. 构造出一个QuoteResponse
4. 将QuoteResponse打印到控制台上

参考文章：

- http://ifeve.com/introducing-actors-akka-notes-part-1/
- http://ifeve.com/akka-notes-actor-messaging-1/
- http://ifeve.com/akka-notes-logging-and-testing/
- http://ifeve.com/akka-notes-actor-messaging-request-and-response-3/
- http://rerun.me/2014/09/11/introducing-actors-akka-notes-part-1/
- http://rerun.me/2014/09/19/akka-notes-actor-messaging-1/
- http://rerun.me/2014/09/29/akka-notes-logging-and-testing/
- http://rerun.me/2014/10/06/akka-notes-actor-messaging-request-and-response-3/
- http://www.oschina.net/p/akka
