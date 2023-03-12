---
title: "MySQL学习——MySQL的Online DDL处理"
date: 2017-12-02 14:51:36
tags: [MySQL]
categories: [DB]
---

## MySQL的Online DDL处理

### 环境

- OS : 阿里云CentOS 7
- MySQL : 5.6.35

### 重点关注

- DDL操作的性能和存储
- DDL操作的并发性

### Online DDL选项

#### DDL操作的性能和存储

MySQL 在线DDL分为 INPLACE 和 COPY 两种方式，通过在ALTER语句的ALGORITHM参数指定。

- ALGORITHM=DEFAULT，指定ALGORITHM = DEFAULT与指定无ALGORITHM子句完全相同，在这种情况下，如果存储引擎支持，则使用ALGORITM = INPLACE。否则，使用ALGORITHM = COPY。
- ALGORITHM=INPLACE，指定ALGORITHM = INPLACE将不使用复制原始表的方式，使得操作对支持它的子句和存储引擎使用在原表上进行操作的技术，从而避免冗长的表副本，可以避免重建表带来的IO和CPU消耗，保证ddl期间依然有良好的性能和并发。
- ALGORITHM=COPY，使用ALGORITHM = COPY子句运行的ALTER TABLE操作会阻止并发的DML操作。并发查询仍然是允许的。也就是说，表复制操作总是至少包括LOCK = SHARED（允许查询，而不是DML）的并发限制。您可以通过指定LOCK = EXCLUSIVE来进一步限制这些操作的并发性，从而防止DML和查询。要强制使用ALTER TABLE操作的table-copy表复制方法，否则将不使用它。

#### DDL操作的并发性

要控制正在更改的表的并发读写的级别，请使用LOCK子句。指定此子句的非默认值使您可以在alter操作期间要求一定数量的并发访问或排他性，并且如果所请求的锁定程度不可用，则停止操作。 LOCK子句的参数是：

- LOCK = DEFAULT : 给定ALGORITHM子句（如果有）和ALTER TABLE操作的最大并发级别：允许并发读写（如果受支持）。如果不支持，则允许并发读取。如果不是，则强制执行独占访问。

- LOCK = NONE : 如果支持，则允许并发读取和写入（允许完整的查询和DML访问表）。否则，会发生错误。

- LOCK = SHARED : 如果支持，则允许并发读取但阻止写入（允许查询，但不允许DML）。即使存储引擎对给定的ALGORITHM子句（如果有）和ALTER TABLE操作支持并发写入，写入也会被阻止。如果不支持并发读取，则会发生错误。

- LOCK = EXCLUSIVE : 强制独占访问（完全阻止对表的访问）。即使存储引擎对给定的ALGORITHM子句（如果有）和ALTER TABLE操作支持并发读写，也会执行此操作。

### DDL操作的状态

- In-Place 为Yes是优选项，说明该操作支持INPLACE。
- Rebuilds Table?为No是优选项，因为为Yes需要重建表，大部分情况与In-Place是相反的。
- Allows Concurrent DML？为Yes是优选项，说明ddl期间表依然可读写，可以指定LOCK=NONE（如果操作允许的话mysql自动就是NONE）。
- Only Modifies Metadata?默认所有DDL操作期间都允许查询请求，放在这只是便于参考。
- Notes会对前面几列Yes/No带 * 号的限制说明。

Operation|In-Place?|Rebuilds Table?|Allows Concurrent DML?|Only Modifies Metadata?|Notes|
---|---|---|---|---|---|
CREATE INDEX, ADD INDEX（添加普通索引）|Yes*|No*|Yes|No|对全文索引有限制|
ADD FULLTEXT INDEX（添加全文索引）|Yes*|No*|No|No|如果没有用户定义的FTS_DOC_ID列，添加第一个FULLTEXT索引将重建表。 后续的FULLTEXT索引可能会被添加到同一个表中而不重建表|
DROP INDEX（删除索引）|Yes|No|Yes|Yes|仅修改表的元数据|
OPTIMIZE TABLE（优化表）|Yes*|Yes|Yes|No|从5.6.17开始使用ALGORITHM=INPLACE，当然如果指定了old_alter_table=1或mysqld启动带–skip-new则将还是COPY模式。如果表上有全文索引只支持COPY|
Set column default value（设置列默认值）|Yes|No|Yes|Yes|仅修改表的元数据|
Change auto-increment value（修改auto-increment值）|Yes|No|Yes|No*|修改存储在内存中的值，而不是数据文件|
Add foreign key constraint（添加外键）|Yes*|No|Yes|Yes|当禁用foreign_key_checks时，可以使用in-place算法，否则必须使用copy算法|
Drop foreign key constraint（删除外键）|Yes|No|Yes|Yes|foreign_key_checks参数没有任何影响|
Rename column（改变列名）|Yes*|No*|Yes*|Yes|为了允许DML并发，仅改变列名，不改变数据类型|
Add column（添加列）|Yes|Yes*|Yes*|No|当添加列是auto-increment，不允许DML并发。尽管允许ALGORITHM=INPLACE ，但数据大幅重组，所以它仍然是一项昂贵的操作|
Drop column（删除列）|Yes|Yes|Yes|No|尽管允许ALGORITHM=INPLACE ，但数据大幅重组，所以它仍然是一项昂贵的操作|
Reorder columns（重新排列列）|Yes|Yes|Yes|No|
Change ROW_FORMAT property（更改ROW_FORMAT属性）|Yes|Yes|Yes|No|尽管允许ALGORITHM=INPLACE ，但数据大幅重组，所以它仍然是一项昂贵的操作|
Change KEY_BLOCK_SIZE property（更改KEY_BLOCK_SIZE属性）|Yes|Yes|Yes|No|尽管允许ALGORITHM=INPLACE ，但数据大幅重组，所以它仍然是一项昂贵的操作|
Make column NULL（设置列属性为NULL）|Yes|Yes*|Yes|No|重建表，但数据大幅重组，所以它仍然是一项昂贵的操作|
Make column NOT NULL（设置列属性为NOT NULL）|Yes*|Yes*|Yes|No|重建表，STRICT_ALL_TABLES或STRICT_TRANS_TABLES SQL_MODE是操作成功所必需的。 如果列包含NULL值，则操作失败。 从5.6.7开始，服务器禁止更改可能导致参照完整性丢失的外键列。但数据大幅重组，所以它仍然是一项昂贵的操作|
Change column data type（修改列数据类型）|No|Yes|No|No|只支持ALGORITHM=COPY|
Add primary key（添加主键）|Yes*|Yes*|Yes|No|尽管允许ALGORITHM=INPLACE ，但数据大幅重组，所以它仍然是一项昂贵的操作。如果列定义必须转化NOT NULL，则不允许INPLACE|
Drop primary key and add another（删除并添加主键）|Yes|Yes|Yes|No|在同一个 ALTER TABLE 语句删除旧主键、添加新主键时，才允许inplace；数据大幅重组,所以它仍然是一项昂贵的操作|
Drop primary key（删除主键）|No|Yes|No|No|只有ALGORITHM=COPY支持在同一个操作中删除一个旧主键而不添加新主键|
Convert character set（转换表字符集）|No|Yes*|No|No|如果新的字符集编码不同，则重建表|
Specify character set（指定字符集）|No|Yes*|No|No|如果新的字符集编码不同，则重建表|

### Online DDL操作实例

```
DROP TABLE IF EXISTS `test_order`;
CREATE TABLE `test_order` (
  `I_ID` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '本表自增主键',
  `BATCH_ID` varchar(40) DEFAULT NULL COMMENT '批次ID',
  `ORDER_ID` varchar(40) NOT NULL DEFAULT '订单ID',
  `USER_ID` varchar(40) NOT NULL COMMENT '用户ID',
  `D_CREATED_AT` datetime DEFAULT NULL COMMENT '创建时间',
  `D_UPDATED_AT` datetime DEFAULT NULL COMMENT '修改时间',
  `I_AMOUNT` int(11) NOT NULL DEFAULT '-1' COMMENT '金额',
  `CH_COMMENT` varchar(50) NOT NULL DEFAULT '' COMMENT '备注',
  PRIMARY KEY (`I_ID`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8 COMMENT='订单表';
```

```
-- 删除索引
ALTER TABLE `test_order` DROP KEY `UID_CT`;
```

```
mysql> select count(*) from test_order;
+----------+
| count(*) |
+----------+
| 10102938 |
+----------+
1 row in set (4.57 sec)
```

数据库test_order表中有大概1000w的数据，分别使用copy和inplace方式进行索引创建。

使用show processlist查看当前执行情况，在创建索引的同时执行insert语句，只有inplace方式可以insert成功，而copy方式需要等待索引创建完成之后才会执行insert语句。

#### 执行效率

##### 使用copy方式

```
mysql> show index from test_order;
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| test_order |          0 | PRIMARY  |            1 | I_ID        | A         |     8865151 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

-- 使用copy方式，使用只读锁
mysql> ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`), algorithm=copy, lock=shared;
Query OK, 10102938 rows affected (2 min 16.04 sec)
Records: 10102938  Duplicates: 0  Warnings: 0
```


##### 使用inplace方式

```
mysql> show index from test_order;
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| test_order |          0 | PRIMARY  |            1 | I_ID        | A         |    10019670 |     NULL | NULL   |      | BTREE      |         |               |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
1 row in set (0.00 sec)

-- 使用inplace方式，不使用锁
mysql> ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`), algorithm=inplace, lock=none;
Query OK, 0 rows affected (47.47 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

执行效率上，创建索引inplace方式要比copy方式要快（表的数据量越大效果越明显）。

#### 并发性

##### copy方式

```
mysql> show processlist;
+-----+------+------------------+-----------------+---------+-------+---------------------------------+------------------------------------------------------------------------------------------------------+
| Id  | User | Host             | db              | Command | Time  | State                           | Info                                                                                                 |
+-----+------+------------------+-----------------+---------+-------+---------------------------------+------------------------------------------------------------------------------------------------------+                                                                                
| 287 | root | 172.18.0.1:38644 | broker_20171130 | Query   |     6 | Waiting for table metadata lock | insert into test_order (BATCH_ID, ORDER_ID, USER_ID, D_CREATED_AT, D_UPDATED_AT, I_AMOUNT, CH_COMMEN |
| 288 | root | 172.18.0.1:38646 | broker_20171130 | Query   |     9 | copy to tmp table               | ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`), algorithm=copy, lock=shared  |
| 289 | root | localhost        | broker_20171130 | Query   |     0 | init                            | show processlist                                                                                     |
+-----+------+------------------+-----------------+---------+-------+---------------------------------+------------------------------------------------------------------------------------------------------+
3 rows in set (0.00 sec)
```

在创建索引的同时，执行insert语句向test_order表中插入数据是会被阻塞的。通过show processlist可以看出，使用copy方式创建索引时，指定lock为shared是允许并发读取但会阻止写入。

##### inplace方式

```
mysql> show processlist;
+-----+------+------------------+-----------------+---------+-------+----------------+------------------------------------------------------------------------------------------------------+
| Id  | User | Host             | db              | Command | Time  | State          | Info                                                                                                 |
+-----+------+------------------+-----------------+---------+-------+----------------+------------------------------------------------------------------------------------------------------+
| 289 | root | localhost        | broker_20171130 | Query   |     0 | init           | show processlist                                                                                     |
| 290 | root | 172.18.0.1:38648 | broker_20171130 | Query   |     9 | altering table | ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`), algorithm=inplace, lock=none |
+-----+------+------------------+-----------------+---------+-------+----------------+------------------------------------------------------------------------------------------------------+
15 rows in set (0.00 sec)
```

在创建索引的同时，执行insert语句向test_order表中插入数据是不会被阻塞的。通过show processlist可以看出，使用inplace方式创建索引时，指定lock为none是不会锁表的，同时可以执行DML语句。

##### 注意

我这里只尝试了创建索引的情况，MySQL官网提供的alter table的其他情况可以自行手动验证。

### MySQL5.5不支持Online DDL选项

因为Online DDL是MySQL5.6的新特性，所以MySQL5.5是不支持的。

```
mysql> ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`), algorithm=inplace, lock=none;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'algorithm=inplace, lock=none' at line 1

mysql> ALTER TABLE `test_order` ADD INDEX `UID_CT` (`USER_ID`,`D_CREATED_AT`);
Query OK, 0 rows affected (0.15 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

参考文章：

- https://dev.mysql.com/doc/refman/5.6/en/innodb-create-index-overview.html
- https://dev.mysql.com/doc/refman/5.6/en/alter-table.html
- https://dev.mysql.com/doc/refman/5.5/en/alter-table.html
- http://www.ywnds.com/?p=8893
