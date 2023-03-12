---
title: "MySQL学习——记一次MySQL的查询优化"
date: 2017-11-30 21:24:51
tags: [MySQL]
categories: [DB]
---

## MySQL学习——记一次MySQL的查询优化

### 环境

- OS : 阿里云CentOS 7
- MySQL : 5.6.35

### 问题描述

- 线上交易订单查询非常慢，查询条件为完成时间区间，公司ID，商户ID
- 总交易订单的数据量在500w左右

```
# 订单表的表结构
DROP TABLE IF EXISTS `b_user_bills`;
CREATE TABLE `b_user_bills` (
  `I_ID` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '本表自增主键',
  `CH_BATCH_ID` varchar(40) NOT NULL COMMENT '批次ID',
  `CH_ORDER_ID` varchar(40) NOT NULL COMMENT '订单ID',
  `I_REF` bigint(20) NOT NULL COMMENT '流水号',
  `CH_BROKER_ID` varchar(40) NOT NULL COMMENT '公司ID',
  `CH_DEALER_ID` varchar(40) NOT NULL COMMENT '商户ID',
  `CH_ID_CARD` varchar(40) NOT NULL COMMENT '用户身份证号',
  `CH_REAL_NAME` varchar(40) NOT NULL COMMENT '用户姓名',
  `I_AMOUNT` int(11) NOT NULL DEFAULT '0' COMMENT '金额',
  `I_STATUS` tinyint(4) NOT NULL DEFAULT '0' COMMENT '状态',
  `D_CREATED_AT` datetime DEFAULT NULL COMMENT '创建时间',
  `D_UPDATED_AT` datetime DEFAULT NULL COMMENT '修改时间',
  `D_FINISHED_TIME` datetime DEFAULT NULL COMMENT '完成时间',
  PRIMARY KEY (`I_ID`),
  UNIQUE KEY `I_REF` (`I_REF`),
  KEY `ORDER_DID` (`CH_ORDER_ID`, `CH_DEALER_ID`),
  KEY `INDEX_BDB` (`CH_BROKER_ID`, `CH_DEALER_ID`, `CH_BATCH_ID`),
  KEY `CH_BROKER_ID` (`CH_BROKER_ID`),
  KEY `CH_DEALER_ID` (`CH_DEALER_ID`),
  KEY `CH_BATCH_ID` (`CH_BATCH_ID`),
  KEY `CH_ID_CARD` (`CH_ID_CARD`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8 COMMENT='订单表';

# 现在按照完成时间区间的条件进行查询非常慢，查询5条记录需要2秒多
mysql> select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| I_ID    | CH_BATCH_ID | CH_ORDER_ID                | I_REF              | CH_BROKER_ID | CH_DEALER_ID | CH_ID_CARD         | CH_REAL_NAME | I_AMOUNT | I_STATUS | D_CREATED_AT        | D_UPDATED_AT        | D_FINISHED_TIME     |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| 1057314 |             | 1000003648282513818534XXXX | 101802225333305728 | yxxxxx3      | hxxxxxn      | 370123XXXXXXXXXXXX | 杨XXXX       |      100 |        1 | 2017-10-31 05:52:30 | 2017-10-31 05:52:31 | 2017-11-01 05:14:33 |
| 1058130 |             | 1000003648283027104438XXXX | 101804732212183424 | yxxxxx3      | hxxxxxn      | 320382XXXXXXXXXXXX | 刘XX         |     3900 |        1 | 2017-10-31 06:12:25 | 2017-10-31 06:12:26 | 2017-11-01 04:43:16 |
| 1059648 |             | 1000003648283740578231XXXX | 101808215636181376 | yxxxxx3      | hxxxxxn      | 232330XXXXXXXXXXXX | 王XX         |      300 |        1 | 2017-10-31 06:40:06 | 2017-10-31 06:40:07 | 2017-11-01 05:45:19 |
| 1059690 |             | 1000003648283759907648XXXX | 101808309858861440 | yxxxxx3      | hxxxxxn      | 330781XXXXXXXXXXXX | 陈XX         |      200 |        1 | 2017-10-31 06:40:51 | 2017-10-31 06:40:52 | 2017-11-01 06:21:07 |
| 1061935 |             | 1000003648284744186645XXXX | 101813115963179392 | yxxxxx3      | hxxxxxn      | 371324XXXXXXXXXXXX | 孟XXXX       |      100 |        1 | 2017-10-31 07:19:03 | 2017-10-31 07:19:04 | 2017-11-01 06:58:01 |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
5 rows in set (2.29 sec)
```

### 分析方法

- show profile分析查询
- explain分析查询
- optimizer_trace分析查询

#### show profile分析查询

```
# 查看profiling功能状态
mysql> select @@profiling;

# 开启profiling功能
mysql> SET profiling=1;

# 执行分析的查询语句
mysql> select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;

# 查看所有查询语句的profiling结果
mysql> show profiles;
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                                                                                                                |
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|        1 | 0.00015900 | select @@profiling                                                                                                                                                                                   |
|        2 | 2.28744900 | select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5          |
+----------+------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

# 查看Query_ID为2的profiling详情
mysql> show profile all for query 2;
+----------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+-----------------------+------------------+-------------+
| Status               | Duration | CPU_user | CPU_system | Context_voluntary | Context_involuntary | Block_ops_in | Block_ops_out | Messages_sent | Messages_received | Page_faults_major | Page_faults_minor | Swaps | Source_function       | Source_file      | Source_line |
+----------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+-----------------------+------------------+-------------+
| starting             | 0.000064 | 0.000058 |   0.000007 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | NULL                  | NULL             |        NULL |
| checking permissions | 0.000011 | 0.000004 |   0.000006 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | check_access          | sql_parse.cc     |        5297 |
| Opening tables       | 0.000016 | 0.000015 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | open_tables           | sql_base.cc      |        5089 |
| init                 | 0.000033 | 0.000032 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | mysql_prepare_select  | sql_select.cc    |        1050 |
| System lock          | 0.000007 | 0.000006 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | mysql_lock_tables     | lock.cc          |         304 |
| optimizing           | 0.000013 | 0.000013 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | optimize              | sql_optimizer.cc |         138 |
| statistics           | 0.000146 | 0.000144 |   0.000002 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | optimize              | sql_optimizer.cc |         362 |
| preparing            | 0.000019 | 0.000017 |   0.000002 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | optimize              | sql_optimizer.cc |         485 |
| executing            | 0.000004 | 0.000003 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | exec                  | sql_executor.cc  |         110 |
| Sending data         | 2.287065 | 2.560400 |   0.060402 |                 5 |                3909 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | exec                  | sql_executor.cc  |         190 |
| end                  | 0.000012 | 0.000006 |   0.000006 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | mysql_execute_select  | sql_select.cc    |        1105 |
| query end            | 0.000007 | 0.000005 |   0.000001 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | mysql_execute_command | sql_parse.cc     |        4996 |
| closing tables       | 0.000012 | 0.000010 |   0.000002 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | mysql_execute_command | sql_parse.cc     |        5044 |
| freeing items        | 0.000022 | 0.000010 |   0.000012 |                 0 |                   0 |            0 |             0 |             1 |                 0 |                 0 |                 0 |     0 | mysql_parse           | sql_parse.cc     |        6433 |
| cleaning up          | 0.000018 | 0.000017 |   0.000002 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | dispatch_command      | sql_parse.cc     |        1778 |
+----------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+-----------------------+------------------+-------------+
15 rows in set, 1 warning (0.00 sec)
```

从show profile的结果可以看到查询耗时都在"Sending data"这个阶段，原来这个状态的名称很具有误导性，所谓的"Sending data"并不是单纯的发送数据，而是包括“收集 + 发送数据”。这里的关键是为什么要收集数据，原因在于：MySQL使用“索引”完成查询结束后，MySQL得到了一堆的行id，如果有的列并不在索引中，MySQL需要重新到“数据行”上将需要返回的数据读取出来返回个客户端。所以"Sending data"这个阶段耗时长，可能是因为读取的“数据行”的方式慢，或者读取的“数据行”的数据结果大。

#### explain分析查询

```
# explain分析查询
mysql> explain select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+----+-------------+--------------+------+-------------------------------------+-----------+---------+-------------+---------+------------------------------------+
| id | select_type | table        | type | possible_keys                       | key       | key_len | ref         | rows    | Extra                              |
+----+-------------+--------------+------+-------------------------------------+-----------+---------+-------------+---------+------------------------------------+
|  1 | SIMPLE      | b_user_bills | ref  | INDEX_BDB,CH_BROKER_ID,CH_DEALER_ID | INDEX_BDB | 244     | const,const | 2447239 | Using index condition; Using where |
+----+-------------+--------------+------+-------------------------------------+-----------+---------+-------------+---------+------------------------------------+
1 row in set (0.01 sec)
```

通过explain的分析结果，发现该查询会使用INDEX_BDB的索引，但是D_FINISHED_TIME字段上却没有索引可能是查询慢的原因。

#### 添加D_FINISHED_TIME索引

下面是线上实际查询的情况

- 因为线上交易订单查询要求必须选择时间段，所以D_FINISHED_TIME是一定存在的查询条件
- 商户角色一定带有CH_DEALER_ID条件，CH_BROKER_ID条件是可选的
- 公司角色一定带有CH_BROKER_ID条件，CH_DEALER_ID条件是可选的
- 管理员角色CH_DEALER_ID和CH_BROKER_ID条件都是可选的

我们根据这几种情况，最终决定创建下面两个索引，理论上应该能够满足所有的查询情况都是用这两个索引

- FT_DID_BID索引：D_FINISHED_TIME, CH_DEALER_ID, CH_BROKER_ID
- FT_BID_DID索引：D_FINISHED_TIME, CH_BROKER_ID, CH_DEALER_ID

具体查询条件组合如下：

- 管理员角色
 - D_FINISHED_TIME
 - D_FINISHED_TIME, CH_DEALER_ID
 - D_FINISHED_TIME, CH_BROKER_ID

- 商户角色
 - D_FINISHED_TIME, CH_DEALER_ID
 - D_FINISHED_TIME, CH_DEALER_ID, CH_BROKER_ID

- 公司角色
 - D_FINISHED_TIME, CH_BROKER_ID
 - D_FINISHED_TIME, CH_BROKER_ID, CH_DEALER_ID

这几个查询条件的组合都按照最左原则满足匹配上面的两个索引FT_DID_BID和FT_BID_DID，这里没有考虑拼接SQL的先后顺序，因为MySQL查询优化器会对AND条件连接的SQL进行优化并且重新排序，以满足尽量使用已经存在的索引，后面会详细介绍。

#### 验证索引的使用情况

```
# 添加新索引
mysql> ALTER TABLE `b_user_bills` ADD INDEX `FT_DID_BID` (D_FINISHED_TIME, CH_DEALER_ID, CH_BROKER_ID);
Query OK, 0 rows affected (15.37 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> ALTER TABLE `b_user_bills` ADD INDEX `FT_BID_DID` (D_FINISHED_TIME, CH_BROKER_ID, CH_DEALER_ID);
Query OK, 0 rows affected (15.49 sec)
Records: 0  Duplicates: 0  Warnings: 0

# 再次查询同样的SQL，并没有明显的改善
mysql> select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| I_ID    | CH_BATCH_ID | CH_ORDER_ID                | I_REF              | CH_BROKER_ID | CH_DEALER_ID | CH_ID_CARD         | CH_REAL_NAME | I_AMOUNT | I_STATUS | D_CREATED_AT        | D_UPDATED_AT        | D_FINISHED_TIME     |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| 1057314 |             | 1000003648282513818534XXXX | 101802225333305728 | yxxxxx3      | hxxxxxn      | 370123XXXXXXXXXXXX | 杨XXXX       |      100 |        1 | 2017-10-31 05:52:30 | 2017-10-31 05:52:31 | 2017-11-01 05:14:33 |
| 1058130 |             | 1000003648283027104438XXXX | 101804732212183424 | yxxxxx3      | hxxxxxn      | 320382XXXXXXXXXXXX | 刘XX         |     3900 |        1 | 2017-10-31 06:12:25 | 2017-10-31 06:12:26 | 2017-11-01 04:43:16 |
| 1059648 |             | 1000003648283740578231XXXX | 101808215636181376 | yxxxxx3      | hxxxxxn      | 232330XXXXXXXXXXXX | 王XX         |      300 |        1 | 2017-10-31 06:40:06 | 2017-10-31 06:40:07 | 2017-11-01 05:45:19 |
| 1059690 |             | 1000003648283759907648XXXX | 101808309858861440 | yxxxxx3      | hxxxxxn      | 330781XXXXXXXXXXXX | 陈XX         |      200 |        1 | 2017-10-31 06:40:51 | 2017-10-31 06:40:52 | 2017-11-01 06:21:07 |
| 1061935 |             | 1000003648284744186645XXXX | 101813115963179392 | yxxxxx3      | hxxxxxn      | 371324XXXXXXXXXXXX | 孟XXXX       |      100 |        1 | 2017-10-31 07:19:03 | 2017-10-31 07:19:04 | 2017-11-01 06:58:01 |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
5 rows in set (2.08 sec)

# explain分析之后，发现该查询并没有使用我们新创建的两个索引，仍然使用的INDEX_BDB索引
mysql> explain select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
| id | select_type | table        | type | possible_keys                                             | key       | key_len | ref         | rows    | Extra                              |
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
|  1 | SIMPLE      | b_user_bills | ref  | INDEX_BDB,CH_BROKER_ID,CH_DEALER_ID,FT_DID_BID,FT_BID_DID | INDEX_BDB | 244     | const,const | 2444634 | Using index condition; Using where |
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
1 row in set (0.01 sec)
```

explain分析之后，发现该查询并没有如预期的一样使用两个新索引，仍然使用的INDEX_BDB索引，这里主要到Extra中的"Using index condition"，可能是因为该查询使用FT_DID_BID和FT_BID_DID索引没有使用INDEX_BDB索引查询快，所以MySQL查询优化器使用了INDEX_BDB索引。下面我们通过optimizer_trace分析查询具体的优化步骤，分析为什么MySQL会使用INDEX_BDB索引。

#### optimizer_trace分析查询

```
# 开启optimizer_trace分析查询
mysql> show variables like '%trace%';
mysql> set optimizer_trace = "enabled=on";
mysql> set optimizer_trace_max_mem_size=1000000;
mysql> set end_markers_in_json=on;

# 重新执行select查询之后，查看MySQL查询优化器的优化步骤
mysql> select * from information_schema.optimizer_trace\G
*************************** 1. row ***************************
                            QUERY: select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `b_user_bills`.`I_ID` AS `I_ID`,`b_user_bills`.`CH_BATCH_ID` AS `CH_BATCH_ID`,`b_user_bills`.`CH_ORDER_ID` AS `CH_ORDER_ID`,`b_user_bills`.`I_REF` AS `I_REF`,`b_user_bills`.`CH_BROKER_ID` AS `CH_BROKER_ID`,`b_user_bills`.`CH_DEALER_ID` AS `CH_DEALER_ID`,`b_user_bills`.`CH_ID_CARD` AS `CH_ID_CARD`,`b_user_bills`.`CH_REAL_NAME` AS `CH_REAL_NAME`,`b_user_bills`.`I_AMOUNT` AS `I_AMOUNT`,`b_user_bills`.`I_STATUS` AS `I_STATUS`,`b_user_bills`.`D_CREATED_AT` AS `D_CREATED_AT`,`b_user_bills`.`D_UPDATED_AT` AS `D_UPDATED_AT`,`b_user_bills`.`D_FINISHED_TIME` AS `D_FINISHED_TIME` from `b_user_bills` where ((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3')) limit 0,5"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "table_dependencies": [
              {
                "table": "`b_user_bills`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`b_user_bills`",
                "field": "CH_BROKER_ID",
                "equals": "'yxxxxx3'",
                "null_rejecting": false
              },
              {
                "table": "`b_user_bills`",
                "field": "CH_DEALER_ID",
                "equals": "'hxxxxxn'",
                "null_rejecting": false
              },
              {
                "table": "`b_user_bills`",
                "field": "CH_BROKER_ID",
                "equals": "'yxxxxx3'",
                "null_rejecting": false
              },
              {
                "table": "`b_user_bills`",
                "field": "CH_DEALER_ID",
                "equals": "'hxxxxxn'",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`b_user_bills`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 4889269,
                    "cost": 5.87e6
                  } /* table_scan */,
                  "potential_range_indices": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "I_REF",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "ORDER_DID",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "INDEX_BDB",
                      "usable": true,
                      "key_parts": [
                        "CH_BROKER_ID",
                        "CH_DEALER_ID",
                        "CH_BATCH_ID",
                        "I_ID"
                      ] /* key_parts */
                    },
                    {
                      "index": "CH_BROKER_ID",
                      "usable": true,
                      "key_parts": [
                        "CH_BROKER_ID",
                        "I_ID"
                      ] /* key_parts */
                    },
                    {
                      "index": "CH_DEALER_ID",
                      "usable": true,
                      "key_parts": [
                        "CH_DEALER_ID",
                        "I_ID"
                      ] /* key_parts */
                    },
                    {
                      "index": "CH_BATCH_ID",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "CH_ID_CARD",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "FT_DID_BID",
                      "usable": true,
                      "key_parts": [
                        "D_FINISHED_TIME",
                        "CH_DEALER_ID",
                        "CH_BROKER_ID",
                        "I_ID"
                      ] /* key_parts */
                    },
                    {
                      "index": "FT_BID_DID",
                      "usable": true,
                      "key_parts": [
                        "D_FINISHED_TIME",
                        "CH_BROKER_ID",
                        "CH_DEALER_ID",
                        "I_ID"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indices */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "INDEX_BDB",
                        "ranges": [
                          "yxxxxx3 <= CH_BROKER_ID <= yxxxxx3 AND hxxxxxn <= CH_DEALER_ID <= hxxxxxn"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 2444634,
                        "cost": 2.93e6,
                        "chosen": true
                      },
                      {
                        "index": "CH_BROKER_ID",
                        "ranges": [
                          "yxxxxx3 <= CH_BROKER_ID <= yxxxxx3"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": true,
                        "index_only": false,
                        "rows": 2444634,
                        "cost": 2.91e6,
                        "chosen": true
                      },
                      {
                        "index": "CH_DEALER_ID",
                        "ranges": [
                          "hxxxxxn <= CH_DEALER_ID <= hxxxxxn"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": true,
                        "index_only": false,
                        "rows": 2444634,
                        "cost": 2.91e6,
                        "chosen": false,
                        "cause": "cost"
                      },
                      {
                        "index": "FT_DID_BID",
                        "ranges": [
                          "2017-11-01 00:00:00 <= D_FINISHED_TIME <= 2017-11-15 23:59:59"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 2444634,
                        "cost": 2.93e6,
                        "chosen": false,
                        "cause": "cost"
                      },
                      {
                        "index": "FT_BID_DID",
                        "ranges": [
                          "2017-11-01 00:00:00 <= D_FINISHED_TIME <= 2017-11-15 23:59:59"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 2444634,
                        "cost": 2.93e6,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "intersecting_indices": [
                        {
                          "index": "CH_BROKER_ID",
                          "index_scan_cost": 38198,
                          "cumulated_index_scan_cost": 38198,
                          "disk_sweep_cost": 170240,
                          "cumulated_total_cost": 208438,
                          "usable": true,
                          "matching_rows_now": 2.44e6,
                          "isect_covering_with_this_index": false,
                          "chosen": true
                        },
                        {
                          "index": "CH_DEALER_ID",
                          "index_scan_cost": 38198,
                          "cumulated_index_scan_cost": 76397,
                          "disk_sweep_cost": 170110,
                          "cumulated_total_cost": 246507,
                          "usable": true,
                          "matching_rows_now": 1.22e6,
                          "isect_covering_with_this_index": false,
                          "chosen": false,
                          "cause": "does_not_reduce_cost"
                        }
                      ] /* intersecting_indices */,
                      "clustered_pk": {
                        "clustered_pk_added_to_intersect": false,
                        "cause": "no_clustered_pk_index"
                      } /* clustered_pk */,
                      "chosen": false,
                      "cause": "too_few_indexes_to_merge"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "CH_BROKER_ID",
                      "rows": 2444634,
                      "ranges": [
                        "yxxxxx3 <= CH_BROKER_ID <= yxxxxx3"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 2444634,
                    "cost_for_plan": 2.91e6,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`b_user_bills`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "INDEX_BDB",
                      "rows": 2.44e6,
                      "cost": 616607,
                      "chosen": true
                    },
                    {
                      "access_type": "ref",
                      "index": "CH_BROKER_ID",
                      "rows": 2.44e6,
                      "cost": 616607,
                      "chosen": false
                    },
                    {
                      "access_type": "ref",
                      "index": "CH_DEALER_ID",
                      "rows": 2.44e6,
                      "cost": 616607,
                      "chosen": false
                    },
                    {
                      "access_type": "range",
                      "rows": 1.83e6,
                      "cost": 3.39e6,
                      "chosen": false
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "cost_for_plan": 616607,
                "rows_for_plan": 2.44e6,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`b_user_bills`",
                  "attached": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59') and (`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))"
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "refine_plan": [
              {
                "table": "`b_user_bills`",
                "pushed_index_condition": "((`b_user_bills`.`CH_DEALER_ID` = 'hxxxxxn') and (`b_user_bills`.`CH_BROKER_ID` = 'yxxxxx3'))",
                "table_condition_attached": "((`b_user_bills`.`D_FINISHED_TIME` >= '2017-11-01 00:00:00') and (`b_user_bills`.`D_FINISHED_TIME` <= '2017-11-15 23:59:59'))"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.00 sec)
```

#### 优化过程参数描述

主要分为逻辑优化和物理优化两部分

- condition_processing : 逻辑优化，进行where条件处理，分别是相等处理，常量处理，删除冗余条件
 - equality_propagation : 条件化简，等式处理
 - constant_propagation : 条件化简，常量处理
 - trivial_condition_removal : 条件化简，条件去除
- table_dependencies : 逻辑优化，找出表之间的相互依赖关系。非直接可用的优化方式
- ref_optimizer_key_uses : 逻辑优化，找出备选的索引
- rows_estimation : 逻辑优化, 估算每个表的元组个数。单表上进行全表扫描和索引扫描的代价估算。每个索引都估算索引扫描代价。先处理访问类型（explain select_type字段的值），候选项分别为全表扫描和所有的索引，开销最小的那个胜出。如果你的语句有Group By，那么在group_index_range子阶段确定是否有适用于range 访问的索引。
 - table_scan : 全表扫描的信息，单表上进行全表扫描的代价
 - potential_range_indices : 列出备选的索引
 - setup_range_conditions : 如果有可下推的条件，则带条件考虑范围查询
 - group_index_range : 如带有GROUPBY或DISTINCT，则考虑是否有索引可优化这种操作。并考虑带有MIN/MAX的情况
 - analyzing_range_alternatives : 开始计算每个索引做范围扫描的花费（等值比较是范围扫描的特例）
 - chosen_range_access_summary : 开始计算每个索引做范围扫描的花费。总结本阶段最优的
- considered_execution_plans : 物理优化，可考虑的执行计划，开始多表连接的物理优化计算
 - best_access_path : 汇总在considered_access_paths阶段得到的结果
  - considered_access_paths : 可考虑的访问路径
  - access_type : 计算INDEX_BDB索引上使用ref方查找的花费
- attaching_conditions_to_tables : 逻辑优化，尽量把条件绑定到对应的表上，分析where条件是否可以执行pushdown，应该是再扫描该表时过滤掉
- refine_plan : 逻辑优化，下推索引条件"pushed_index_condition"；其他条件附加到表上做为过滤条件"table_condition_attached"
 - pushed_index_condition : 
 - table_condition_attached : 

#### 过程解析

access_type : 就是MySQL在表里找到所需行的方式。包括（由左至右，由最差到最好）：

```
| All | index | range | ref | eq_ref | const, system | null |
```

通过rows_estimation，可以看出INDEX_BDB，CH_BROKER_ID，CH_DEALER_ID，FT_DID_BID，FT_BID_DID这几个索引可以被使用。

通过analyzing_range_alternatives，可以看出所有可以被使用的索引预估的代价，其中CH_BROKER_ID和CH_DEALER_ID的索引代价最小，没有选择使用FT_DID_BID和FT_BID_DID索引是因为cost代价过高。

通过intersecting_indices，可以看出MySQL尝试对多个索引交集访问(intersections)，即对CH_BROKER_ID和CH_DEALER_ID索引交集访问，但是发现cost代价没有降低"does_not_reduce_cost"，所以不进行优化，原因是"too_few_indexes_to_merge"。

通过chosen_range_access_summary，可以看出逻辑优化最优的索引是CH_BROKER_ID，这里其实CH_BROKER_ID和CH_DEALER_ID索引的cost是一样的，但是MySQL优化器会选择第一个索引，即CH_BROKER_ID。

通过considered_access_paths，可以看出ref类型的INDEX_BDB，CH_BROKER_ID，CH_DEALER_ID索引和range索引的cost代价，最终选择cost代价最小的INDEX_BDB索引。

#### 分析原因

```
mysql> select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| I_ID    | CH_BATCH_ID | CH_ORDER_ID                | I_REF              | CH_BROKER_ID | CH_DEALER_ID | CH_ID_CARD         | CH_REAL_NAME | I_AMOUNT | I_STATUS | D_CREATED_AT        | D_UPDATED_AT        | D_FINISHED_TIME     |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| 1057314 |             | 1000003648282513818534XXXX | 101802225333305728 | yxxxxx3      | hxxxxxn      | 370123XXXXXXXXXXXX | 杨XXXX       |      100 |        1 | 2017-10-31 05:52:30 | 2017-10-31 05:52:31 | 2017-11-01 05:14:33 |
| 1058130 |             | 1000003648283027104438XXXX | 101804732212183424 | yxxxxx3      | hxxxxxn      | 320382XXXXXXXXXXXX | 刘XX         |     3900 |        1 | 2017-10-31 06:12:25 | 2017-10-31 06:12:26 | 2017-11-01 04:43:16 |
| 1059648 |             | 1000003648283740578231XXXX | 101808215636181376 | yxxxxx3      | hxxxxxn      | 232330XXXXXXXXXXXX | 王XX         |      300 |        1 | 2017-10-31 06:40:06 | 2017-10-31 06:40:07 | 2017-11-01 05:45:19 |
| 1059690 |             | 1000003648283759907648XXXX | 101808309858861440 | yxxxxx3      | hxxxxxn      | 330781XXXXXXXXXXXX | 陈XX         |      200 |        1 | 2017-10-31 06:40:51 | 2017-10-31 06:40:52 | 2017-11-01 06:21:07 |
| 1061935 |             | 1000003648284744186645XXXX | 101813115963179392 | yxxxxx3      | hxxxxxn      | 371324XXXXXXXXXXXX | 孟XXXX       |      100 |        1 | 2017-10-31 07:19:03 | 2017-10-31 07:19:04 | 2017-11-01 06:58:01 |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
5 rows in set (2.08 sec)

mysql> select * from b_user_bills force index (FT_DID_BID) where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| I_ID    | CH_BATCH_ID | CH_ORDER_ID                | I_REF              | CH_BROKER_ID | CH_DEALER_ID | CH_ID_CARD         | CH_REAL_NAME | I_AMOUNT | I_STATUS | D_CREATED_AT        | D_UPDATED_AT        | D_FINISHED_TIME     |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
| 1057314 |             | 1000003648282513818534XXXX | 101802225333305728 | yxxxxx3      | hxxxxxn      | 370123XXXXXXXXXXXX | 杨XXXX       |      100 |        1 | 2017-10-31 05:52:30 | 2017-10-31 05:52:31 | 2017-11-01 05:14:33 |
| 1058130 |             | 1000003648283027104438XXXX | 101804732212183424 | yxxxxx3      | hxxxxxn      | 320382XXXXXXXXXXXX | 刘XX         |     3900 |        1 | 2017-10-31 06:12:25 | 2017-10-31 06:12:26 | 2017-11-01 04:43:16 |
| 1059648 |             | 1000003648283740578231XXXX | 101808215636181376 | yxxxxx3      | hxxxxxn      | 232330XXXXXXXXXXXX | 王XX         |      300 |        1 | 2017-10-31 06:40:06 | 2017-10-31 06:40:07 | 2017-11-01 05:45:19 |
| 1059690 |             | 1000003648283759907648XXXX | 101808309858861440 | yxxxxx3      | hxxxxxn      | 330781XXXXXXXXXXXX | 陈XX         |      200 |        1 | 2017-10-31 06:40:51 | 2017-10-31 06:40:52 | 2017-11-01 06:21:07 |
| 1061935 |             | 1000003648284744186645XXXX | 101813115963179392 | yxxxxx3      | hxxxxxn      | 371324XXXXXXXXXXXX | 孟XXXX       |      100 |        1 | 2017-10-31 07:19:03 | 2017-10-31 07:19:04 | 2017-11-01 06:58:01 |
+---------+-------------+----------------------------+--------------------+--------------+--------------+--------------------+--------------+----------+----------+---------------------+---------------------+---------------------+
5 rows in set (0.18 sec)
```

使用force index之后，发现明显使用FT_DID_BID索引的查询更快。对比一下这两个SQL的explain查询计划

```
mysql> explain select * from b_user_bills where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
| id | select_type | table        | type | possible_keys                                             | key       | key_len | ref         | rows    | Extra                              |
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
|  1 | SIMPLE      | b_user_bills | ref  | INDEX_BDB,CH_BROKER_ID,CH_DEALER_ID,FT_DID_BID,FT_BID_DID | INDEX_BDB | 244     | const,const | 2444634 | Using index condition; Using where |
+----+-------------+--------------+------+-----------------------------------------------------------+-----------+---------+-------------+---------+------------------------------------+
1 row in set (0.00 sec)

mysql> explain select * from b_user_bills force index (FT_DID_BID) where d_finished_time >= '2017-11-01 00:00:00' and d_finished_time <= '2017-11-15 23:59:59' and ch_dealer_id = 'hxxxxxn' and ch_broker_id = 'yxxxxx3' limit 0, 5;
+----+-------------+--------------+-------+---------------+------------+---------+------+---------+-----------------------+
| id | select_type | table        | type  | possible_keys | key        | key_len | ref  | rows    | Extra                 |
+----+-------------+--------------+-------+---------------+------------+---------+------+---------+-----------------------+
|  1 | SIMPLE      | b_user_bills | range | FT_DID_BID    | FT_DID_BID | 250     | NULL | 2444634 | Using index condition |
+----+-------------+--------------+-------+---------------+------------+---------+------+---------+-----------------------+
1 row in set (0.00 sec)
```

explain不同的地方：

- type : MySQL 在表里找到所需行的方式，All < index < range < ref < eq_ref < const, system < null
 - ref : 扫描非唯一性索引，返回匹配某个单独值的所有行。
 - range : 以范围的形式扫描索引。扫描部分索引，索引范围扫描，对索引的扫描开始于某一点，返回匹配值域的行。
- key : 查询过程中实际使用的索引
 - INDEX_BDB
 - FT_DID_BID
- key_len : 索引字段最大可能使用的长度。用于表示本次查询中，所选择的索引长度有多少字节，通常我们可借此判断联合索引有多少列被选择了。
 - 244 : CH_BATCH_ID，CH_DEALER_ID和CH_BROKER_ID都为varchar(40)且不允许为NULL，但因为查询条件只使用到了CH_DEALER_ID和CH_BROKER_ID，所以key_len = 40 * 3 + 2 + 40 * 3 + 2 = 244
 - 250 : D_FINISHED_TIME为datetime且允许为NULL，CH_DEALER_ID和CH_BROKER_ID都为varchar(40)且不允许为NULL，而且查询条件D_FINISHED_TIME，CH_DEALER_ID和CH_BROKER_ID都使用到了，所以key_len = 5 + 1 + 40 * 3 + 2 + 40 * 3 + 2 = 250

> #### key_len 大小的计算规则：
> 
> - 一般地，key_len 等于索引列类型字节长度，例如int类型为4-bytes，bigint为8-bytes
> 
> - 如果是字符串类型，还需要同时考虑字符集因素，例如：CHAR(30) UTF8则key_len至少是90-bytes
> 
> - 若该列类型定义时允许NULL，其key_len还需要再加 1-bytes
> 
> - 若该列类型为变长类型，例如 VARCHAR（TEXT\BLOB不允许整列创建索引，如果创建部分索引，也被视为动态列类型），其key_len还需要再加 2-bytes
>
> 注意：日期时间型的key_len计算：（针对mysql5.6及之后版本）
> 
> - DATETIME允许为NULL =  5 + 1(NULL)
> 
> - DATETIME不允许为NULL = 5
> 
> - TIMESTAMP允许为NULL = 4 + 1(NULL)
> 
> - TIMESTAMP不允许为NULL = 4

从explain和optimizer_trace的分析结果中可以看出，MySQL查询优化器认为使用ref效果更好，是因为cost代价比较小。但是实际执行时，指定使用range方式扫描索引执行效率会更高一些。但是通常情况下，ref的效率是要比range的效率要高的，所以MySQL优先使用ref方式（这是一条启发式规则）。但最终是否使用ref方式还是range方式，MySQL还需要通过代价估算进行比较再做决定。

但是我这里有个疑惑，为什么MySQL查询优化器得到的结果是，INDEX_BDB索引的cost代价最小呢？而不是我新建的索引FT_DID_BID和FT_BID_DID的cost代价最小呢？

### 代价估算

代价估算是一个求近似值的过程，因为计算基于的一些值是估算得来的，并不十分精准，这就造成了计算误差。MySQL的代价估算是基于IO和CPU的代价来估算是否用索引，或者用哪种索引，而这个估算基于的统计信息有可能不准确。

但究竟是否使用ref或range，MySQL还需要通过代价估算进行比较再做决定。

代价估算是一个求近似值的过程，因为计算基于的一些值是估算得来的，并不十分精准，这就造成了计算误差。但是，如果索引的选择率较低（如低于10%），则使用ref的效果好于range的效果的概率大。反过来说，如果索引的选择率较高，则ref未必range的效果好，但是因计算误差，使得执行计划得到了ref好于range的错误结论。

进一步讲，如果索引的选择率很高（如远高于10%，这是大概值，不精确），甚至数据存放是顺序连续的，有可能的是，尽管索引存在，但索引扫描的效果还差与全表扫描。

所以MySQL代价估算是取决于，索引选择率、索引扫描的方式（ref还是range）、使用索引还是全表扫描。

### cardinality选择率

http://blog.csdn.net/alongken2005/article/details/6394016
https://yq.aliyun.com/articles/27296
http://www.penglixun.com/tech/database/mysql_show_index_cardinality.html
http://blog.csdn.net/zheng0518/article/details/50561761
https://www.percona.com/blog/2008/09/03/analyze-myisam-vs-innodb/


### eq_range_index_dive_limit

估计方法有2种:

- index dive: dive到index中即利用索引完成元组数的估算
 - 速度慢，但能得到精确的值（MySQL的实现是数索引对应的索引项个数，所以精确）
- index statistics: 使用索引的统计数值，进行估算
 - 速度快，但得到的值未必精确

> 注意：
> 
> 选项 eq_range_index_dive_limit 的值设定了in列表中的条件个数上线，超过设定值时，会将执行计划从index statistics变成index dive。
> 
> 1. eq_range_index_dive_limit = 0 只能使用index dive
> 2. 0 < eq_range_index_dive_limit <= N 使用index statistics
> 3. eq_range_index_dive_limit > N 只能使用index dive
> 
> ```
> # 可以查看eq_range_index_dive_limit设置的值
> mysql> show variables like 'eq_range_index_dive_limit';
> +---------------------------+-------+
> | Variable_name             | Value |
> +---------------------------+-------+
> | eq_range_index_dive_limit | 10    |
> +---------------------------+-------+
> 1 row in set (0.00 sec)
> ```

一般在更新时间戳上会有索引，但是有时候mysql会判断出某一天的更新量特别大，比如超过了20%，那么根据数据的选择性，mysql决定不用索引。

但是，如果索引的选择率较低（如低于10%），则使用ref的效果好于range的效果的概率大。反过来说，如果索引的选择率较高，则ref未必range的效果好，但是因计算误差，使得执行计划得到了ref好于range的错误结论。
进一步讲，如果索引的选择率很高（如远高于10%，这是大概值，不精确），甚至数据存放是顺序连续的，有可能的是，尽管索引存在，但索引扫描的效果还差与全表扫描。

我们都知道索引是用来加快查询速度的，但是当一个索引辨识度不是很高，或者扫描索引的数据量很大，MySQL查询优化器会选择使用MySQL查询优化器认为合适的索引或者全表扫描来获取数据。前面也提到过，当MySQL使用索引扫描完成查询结束后，MySQL得到了一堆的行id，如果有的列并不在索引中，MySQL需要重新到“数据行”上将需要返回的数据读取出来返回个客户端。MySQL查询优化器会对这个扫描索引的过程进行“预估算”扫描的行数和cost代价，最终选择cost代价最小的索引或者全表扫描来获取数据。



统计信息在不断变化，而且统计信息也无法保证绝对准确，不是MySQL的优化器做得挫，这是所有数据库优化器都面临的一个难题，基于代价计算的模型就那样，怎么把统计信息做得更准确又不影响读写性能？噢，又到了鱼和熊掌不可兼得的哲学高度了，我讨厌这样的问题。所以各家数据库的优化器现在基本PK的是启发式优化，这个是各自的看家本领。DBA和业务的同学是很害怕选错索引的或者统计信息变了，索引选择也变了，至少阿里的DBA和业务同学是很害怕的，各家数据库都支持hint来固定索引

https://www.cnblogs.com/hellohell/p/5718238.html

### ref优化成range

- ref 与 range 使用的是相同的索引；
- 当前 table 选择的索引采用的是ref；
- ref key 的使用的长度小于 range 的长度，则优先使用 range。

参考文章：

- https://www.percona.com/blog/2008/09/03/analyze-myisam-vs-innodb/
- http://imysql.com/2015/10/20/mysql-faq-key-len-in-explain.shtml
- http://www.cnblogs.com/xuanzhi201111/p/4554769.html