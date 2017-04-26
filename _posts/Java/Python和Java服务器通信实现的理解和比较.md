---
title: "Python和Java服务器通信实现的理解和比较"
date: 2017-03-05 18:41:38
tags: [Java Web，Socket，Python]
categories: [Java，Python]
---

### Python的WSGI和Java的Servlet API

#### Python的WSGI

最近在学习使用Python进行WebServer的编程，发现WSGI（Web Server Gateway Interface）的概念。PythonWeb服务器网关接口（Python Web Server Gateway Interface，缩写为WSGI）是Python应用程序或框架和Web服务器之间的一种接口，已经被广泛接受，它已基本达成它的可移植性方面的目标。WSGI 没有官方的实现，因为WSGI更像一个协议。只要遵照这些协议，WSGI应用(Application)都可以在任何服务器(Server)上运行，反之亦然。

如果没有WSGI，你选择的Python网络框架将会限制所能够使用的 Web 服务器。

这就意味着，你基本上只能使用能够正常运行的服务器与框架组合，而不能选择你希望使用的服务器或框架。

那么，你怎样确保可以在不修改 Web 服务器代码或网络框架代码的前提下，使用自己选择的服务器，并且匹配多个不同的网络框架呢？为了解决这个问题，就出现了PythonWeb 服务器网关接口（Web Server Gateway Interface，WSGI）。

WSGI的出现，让开发者可以将网络框架与 Web 服务器的选择分隔开来，不再相互限制。现在，你可以真正地将不同的 Web 服务器与网络开发框架进行混合搭配，选择满足自己需求的组合。例如，你可以使用Gunicorn或Nginx/uWSGI或Waitress服务器来运行Django、Flask或Pyramid应用。正是由于服务器和框架均支持WSGI，才真正得以实现二者之间的自由混合搭配。

#### Java的Servlet API

下面将类比Java来说明一下：

如果没有Java Servlet API，你选择的Java Web容器（Java Socket编程框架实现）将会限制所能够使用的Java Web框架（因为没有Java Servlet API，那么SpringMVC可能会实现一套SpringMVCHttpRequest和SpringMVCHttpResponse标准，Struts2可能会实现一套Struts2HttpRequest和Struts2HttpResponse标准，如果Tomcat只支持SpringMVC的API，那么选择Tomcat服务器就只能使用SpringMVC的Web框架来写服务端代码）。

这就意味着，你基本上只能使用能够正常运行的服务器（Tomcat）与框架（SpringMVC）组合，而不能选择你希望使用的服务器或框架（比如：我要换成Tomcat + Struts2的组合）。

注意：这里假设没有Java Servlet API，这样就相当于SpringMVC和Struts2可能都要自己实现一套Servlet封装HttpRequest和HttpResponse，这样从SpringMVC更换成Struts2就几乎需要重写服务器端的代码。为了解决这个问题，Java提出了Java Servlet API协议，让所有的Web服务框架都实现此Java Servlet API协议来和Java Web服务器（例如：Tomcat）交互，而复杂的网络连接控制等等都交由Java Web服务器来控制，Java Web服务器用Java Socket编程实现了复杂的网络连接管理。


#### 详细说说Python的WSGI

Python Web 开发中，服务端程序可以分为两个部分，一是服务器程序，二是应用程序。前者负责把客户端请求接收，整理，后者负责具体的逻辑处理。为了方便应用程序的开发，我们把常用的功能封装起来，成为各种Web开发框架，例如 Django, Flask, Tornado。不同的框架有不同的开发方式，但是无论如何，开发出的应用程序都要和服务器程序配合，才能为用户提供服务。这样，服务器程序就需要为不同的框架提供不同的支持。这样混乱的局面无论对于服务器还是框架，都是不好的。对服务器来说，需要支持各种不同框架，对框架来说，只有支持它的服务器才能被开发出的应用使用。

这时候，标准化就变得尤为重要。我们可以设立一个标准，只要服务器程序支持这个标准，框架也支持这个标准，那么他们就可以配合使用。一旦标准确定，双方各自实现。这样，服务器可以支持更多支持标准的框架，框架也可以使用更多支持标准的服务器。

- 服务器端：

服务器必须将可迭代对象的内容传递给客户端，可迭代对象会产生bytestrings，必须完全完成每个bytestring后才能请求下一个。

- 应用程序：

服务器程序会在每次客户端的请求传来时，调用我们写好的应用程序，并将处理好的结果返回给客户端。

总结：

- Web Server Gateway Interface是Python编写Web业务统一接口。
- 无论多么复杂的Web应用程序，入口都是一个WSGI处理函数。
- Web应用程序就是写一个WSGI的处理函数，主要功能在于交互式地浏览和修改数据，生成动态Web内容，针对每个HTTP请求进行响应。

#### 实现Python的Web应用程序能被访问的方式

要使 Python 写的程序能在 Web 上被访问，还需要搭建一个支持 Python 的 HTTP 服务器（也就是实现了WSGI server（WSGI协议）的Http服务器）。有如下几种方式：

- 可以自己使用Python Socket编程实现一个Http服务器
- 使用支持Python的开源的Http服务器（如：uWSGI，wsgiref，Mod_WSGI等等）。如果是使用Nginx，Apache，Lighttpd等Http服务器需要单独安装支持WSGI server的模块插件。
- 使用Python开源Web框架（如：Flask，Django等等）内置的Http服务器（Django自带的WSGI Server，一般测试使用）

##### Python标准库对WSGI的实现

wsgiref 是Python标准库给出的 WSGI 的参考实现。simple_server 这一模块实现了一个简单的 HTTP 服务器。

Python源码中的wsgiref的simple_server.py正好说明上面的分工情况，server的主要作用是接受client的请求，並把的收到的请求交給RequestHandlerClass处理，RequestHandlerClass处理完成后回传结果给client

##### uWSGI服务器

uWSGI是一个Web服务器，它实现了WSGI协议、uwsgi、http等协议。注意uwsgi是一种通信协议，而uWSGI是实现uwsgi协议和WSGI协议的Web服务器。

##### Django框架内置的WSGI Server服务器

Django的WSGIServer继承自wsgiref.simple_server.WSGIServer，而WSGIRequestHandler继承自wsgiref.simple_server.WSGIRequestHandler

之前说到的application，在Django中一般是django.core.handlers.wsgi.WSGIHandler对象，WSGIHandler继承自django.core.handlers.base.BaseHandler，这个是Django处理request的核心逻辑，它会创建一个WSGIRequest实例，而WSGIRequest是从http.HttpRequest继承而来

#### Python和Java的类比

##### Python和Java的服务器结构

- 独立WSGI server（实现了Http服务器功能） + Python Web应用程序
 - 例如：Gunicorn，uWSGI + Django，Flask
- 独立Servlet引擎（Java应用服务器）（实现了Http服务器功能） + Java Web应用程序
 - 例如：Jetty，Tomcat + SpringMVC，Struts2

##### Python和Java服务器共同点

- WSGI server（例如Gunicorn和uWSGI）
 - WSGI server服务器内部都有组建来实现Socket连接的创建和管理。
 - WSGI server服务器都实现了Http服务器功能，能接受Http请求，并且通过Python Web应用程序处理之后返回动态Web内容。

- Java应用服务器（Jetty和Tomcat）
 - Java应用服务器内部都有Connector组件来实现Socket连接的创建和管理。
 - Java应用服务器都实现了Http服务器功能，能接受Http请求，并且通过Java Web应用程序处理之后返回动态Web内容。


参考文章：

- http://baike.baidu.com/link?url=xqzfWOAE6U5ZuymeXiaX6VPtfoiYWjtjcfa1uejqEdy0oXlgO8KCra3tu0vU-4k6m9L6BV9l9P4RSDXWBQOjTq
- http://www.kaimingwan.com/post/python/zi-ji-dong-shou-kai-fa-ge-webfu-wu-qi-er
- http://blog.csdn.net/on_1y/article/details/18803563
- http://blog.csdn.net/on_1y/article/details/18818081
- http://www.nowamagic.net/academy/detail/1330328
- http://www.ibm.com/developerworks/cn/java/j-lo-tomcat1/