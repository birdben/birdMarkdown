---
title: "Python学习（二）Python的抽象类和接口"
date: 2017-04-08 16:54:53
tags: [Python]
categories: [Python]
---

### Python的抽象类和接口

Python使用 abc 模块可以很轻松的定义抽象基类，并且可以定义抽象方法（如下：getConn方法），然后子类通过继承DaoSupport基类来实现对应的getConn抽象方法。

```
from abc import abstractmethod, ABCMeta


class DaoSupport(metaclass=ABCMeta):

    def __init__(self):
        pass

    @abstractmethod
    def getConn(self):
        """Abstract Method getConn"""


```

抽象类DaoSupport必须继承ABCMeta。

使用@abstractmethod定义一个抽象方法，该抽象方法不需要任何实现。

```
import time
from pymongo.errors import ConnectionFailure

from dao.support.DaoSupport import DaoSupport
from dao.config import MongoConfig
from logger.LoggingRoot import rootLogger
from pymongo import mongo_client


class MongoDaoSupport(DaoSupport):
    def __init__(self):
        super().__init__()

    def getConn(self):
        rootLogger.debug("MongoDaoSupport getConn start")
        conn = mongo_client.MongoClient(
            host=MongoConfig.HOST,
            port=MongoConfig.PORT,
            document_class=MongoConfig.DOCUMENT_CLASS,
            tz_aware=MongoConfig.TZ_AWARE,
            connect=MongoConfig.CONNECT,
            maxPoolSize=MongoConfig.MAX_POOL_SIZE,
            minPoolSize=MongoConfig.MIN_POOL_SIZE
        )
        # 建立和数据库系统的连接,创建Connection时，指定host及port参数
        db_auth = conn.admin
        # admin 数据库有帐号，连接-认证-切换库
        db_auth.authenticate(MongoConfig.USER, MongoConfig.PWD)
        try:
            conn.admin.command('ismaster')
        except ConnectionFailure:
            print("Server not available")
        rootLogger.debug("MongoDaoSupport getConn end")
        return conn
        
```

如果子类MongoDaoSupport没有实现基类的getConn抽象方法，Python运行时会报错提示子类MongoDaoSupport需要实现基类的getConn抽象方法。

```
Connected to pydev debugger (build 163.10154.50)
Traceback (most recent call last):
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 1596, in <module>
    globals = debugger.run(setup['file'], None, None, is_module)
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 974, in run
    pydev_imports.execfile(file, globals, locals)  # execute the script
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/_pydev_imps/_pydev_execfile.py", line 18, in execfile
    exec(compile(contents+"\n", file, 'exec'), glob, loc)
  File "/Users/yunyu/workspace_python/crawler/dao/support/MongoDaoSupport.py", line 16, in <module>
    dao = MysqlDaoSupport()
TypeError: Can't instantiate abstract class MongoDaoSupport with abstract methods getConn
```


参考文章：

- http://python3-cookbook.readthedocs.io/zh_CN/latest/c08/p12_define_interface_or_abstract_base_class.html
- https://www.smartfile.com/blog/abstract-classes-in-python/