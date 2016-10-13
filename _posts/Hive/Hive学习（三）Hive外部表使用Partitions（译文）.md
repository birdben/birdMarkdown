---
title: "Hive学习（三）Hive外部表使用Partitions（译文）"
date: 2016-10-13 20:36:31
tags: [Hive, HDFS]
categories: [Hadoop]
---

##### 普通的Hive表

可以用下面的script创建普通的Hive表

```
CREATE TABLE user (
  userId BIGINT,
  type INT,
  level TINYINT,
  date String
)
COMMENT 'User Infomation'
```

这个表是没有数据的，直到我们load数据之前这个表是没什么用的

```
LOAD INPATH '/user/chris/data/testdata' OVERWRITE INTO TABLE user
```

默认情况下，当数据文件被加载，/user/${USER}/warehouse/user 会被自动创建。

对我来说，目录是 /user/chris/warehouse/user ，user是表名，user表的数据文件都被定位到这个目录下。

现在，我们可以随意执行SQL来分析数据了。

##### 假如

假如我们想要通过ETL程序处理这些数据，并且加载结果数据到Hive中，但是我们不想手工加载这些结果数据。

假如这些数据不仅仅是被Hive使用，还有一些其他应用程序也使用，可能还会被MapReduce处理。

External Table外部表就是来拯救我们的，通过下面的语法来创建外置表

```
CREATE EXTERNAL TABLE user (
  userId BIGINT,
  type INT,
  level TINYINT,
  date String
)
COMMENT 'User Infomation'
LOCATION '/user/chris/datastore/user/';
```

Location配置是设置我们要将数据文件存储的位置，目录的名称必须和表名一样（就像Hive的普通表一样）。在这个例子中，表名就是user。

然后，我们可以导入任何符合user表声明的pattern表达式的数据文件到user目录下。

所有的数据都可以被Hive SQL立即访问。

##### 不够理想的地方

当数据文件变大（数量和大小），我们可能需要用Partition分区来优化数据处理的效率。

```
CREATE TABLE user (
  userId BIGINT,
  type INT,
  level TINYINT,
)
COMMENT 'User Infomation'
PARTITIONED BY (date String)
```

date String 被移动到 PARTITIONED BY，当我们需要加载数据到Hive时，partition一定要被分配。

```
LOAD INPATH '/user/chris/data/testdata' OVERWRITE INTO TABLE user PARTITION (date='2012-02-22')
```

当数据加载完之后，我们可以看到一个名称是date=2010-02-22的新目录被创建在 /user/chris/warehouse/user/ 下。

所以，我们要如何使用External Table的Partition来优化数据处理呢？

和之前一样，首先要创建外部表user，并且分配好Location。

```
CREATE EXTERNAL TABLE user (
  userId BIGINT,
  type INT,
  level TINYINT,
  date String
)
COMMENT 'User Infomation'
PARTITIONED BY (date String)
LOCATION '/user/chris/datastore/user/';
```

然后，在 /user/chris/datastore/user/ 下创建目录date=2010-02-22

最后，把date是2010-02-22数据文件存储在这个目录下，完成。

但是，

当我们执行select * from user;没有任何结果数据。

为什么呢？

我花了很长时间搜寻答案。

最终，解决了。

因为当外部表被创建，Metastore包含Hive元数据信息，Hive元数据中外置表的默认表路径是被更改到指定的Location，但是关于partition，不做任何更改，所以我们必须手工添加这些元数据。

```
ALTER TABLE user ADD PARTITION(date='2010-02-22');
```

每次有一个新的 date=... 目录（partition）被创建，我们都必须手工alter table来添加partition信息。

这个真的不是很好的方式！

但是幸运的是，我们有Hive JDBC/Thrift, 我们可以使用 [script](https://github.com/don9z/hadoop-tools/blob/master/hive/addpartition.py) 脚本来做这些。

原文链接：

- http://blog.zhengdong.me/2012/02/22/hive-external-table-with-partitions/