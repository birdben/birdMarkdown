---
title: "Akka Helloworld"
date: 2015-11-12 11:51:43
tags: [Akka]
categories: [Akka]
---

功能描述：
实现通过一个actior发送消息到另一个actor然后将处理结果返回，感觉很简单类似两个类的方法调用，但是这里实际上的处理时异步的并非同步的调用处理，这里神奇的地方就在于AKKA的内部机制了后续再做深入研究。

HelloWorld.java

```
package com.birdben.akka.helloworld;

import akka.actor.ActorRef;
import akka.actor.Props;
import akka.actor.UntypedActor;

public class HelloWorld extends UntypedActor {

    @Override
    public void preStart() {
        // create the greeter actor
        final ActorRef greeter =
                getContext().actorOf(Props.create(Greeter.class), "greeter");//创建greeter actor实例
        // tell it to perform the greeting
        greeter.tell(Greeter.Msg.GREET, getSelf());//通过tell方法给greeter actor 发送一条消息
    }

    @Override
    public void onReceive(Object msg) {
        if (msg == Greeter.Msg.DONE) {
            // when the greeter is done, stop this actor and with it the application
            getContext().stop(getSelf());
        } else unhandled(msg);
    }
}  
```

Greeter.java

```
package com.birdben.akka.helloworld;

import akka.actor.UntypedActor;

public class Greeter extends UntypedActor {

    public static enum Msg {
        GREET, DONE;
    }

    @Override
    public void onReceive(Object msg) {
        if (msg == Msg.GREET) {
            System.out.println("Hello World!");
            getSender().tell(Msg.DONE, getSelf());
        } else unhandled(msg);
    }
}
```

运行HelloWorld类，akka提供了一个主actor类，可以通过这个类直接执行以上的方法，在IDEA中点击Edit Configurations（配置tomcat的按钮），新建一个Application，在Main.class 选项中选择akka.Main，然后在program arguments 中输入 com.birdben.akka.helloworld.HelloWorld，点击 apply 然后运行结果如下：

```
com.birdben.akka.helloworld.HelloWorld
Connected to the target VM, address: '127.0.0.1:50084', transport: 'socket'
Hello World!
[INFO] [11/11/2015 01:40:59.782] [Main-akka.actor.default-dispatcher-3] [akka://Main/user/app-terminator] application supervisor has terminated, shutting down
Disconnected from the target VM, address: '127.0.0.1:50084', transport: 'socket'

Process finished with exit code 0

```

从运行结果可以看出，HelloWorld actor正确的调用了 Greeter actor 因为输出了 Hello World！从Infor日志可以看出 HelloWorld actor 正常接收到了 Greeter actor 的返回停止了当前actor。

参考文章：

- http://www.blogbus.com/dreamhead-logs/235916459.html
- http://verran.iteye.com/blog/1942393
