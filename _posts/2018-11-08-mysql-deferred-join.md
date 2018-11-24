---
layout: post
title: MySQL查询优化之延迟关联
description: mysql, 面试, 延迟关联, mysql查询优化, deferred join
categories: [Web]
tags: [MySQL]
shortinfo: MySQL查询优化之延迟关联
---

最近项目上遇到了由于一个分页查询的业务导致请求超时，页面无法正常显示的情况。分析解决问题后，记录下备忘。

> 简单描述下业务，是一张用来记录用户操作行为的表，数据量大概在1500W左右。前端用户通过选择类型，时间等筛选，得到分页结果。

表结构如下
```
| t_operate_6 | CREATE TABLE `t_operate_6` (
  `Fid` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `Fcid` bigint(20) unsigned NOT NULL,
  `Fuid` bigint(20) unsigned NOT NULL,
  `Fname` varchar(32) DEFAULT NULL,
  `FroleName` varchar(64) NOT NULL,
  `Faction` int(2) NOT NULL,
  `FobjectId` varchar(64) DEFAULT NULL,
  `FobjectName` varchar(64) DEFAULT NULL,
  `Fip` varchar(64) DEFAULT NULL,
  `Fdevtype` int(2) NOT NULL DEFAULT '0',
  `Fctime` int(11) unsigned NOT NULL,
  PRIMARY KEY (`Fid`),
  KEY `Fcid` (`Fcid`),
  KEY `Fuid` (`Fuid`),
  KEY `Fctime` (`Fctime`),
) ENGINE=InnoDB AUTO_INCREMENT=14491762 DEFAULT CHARSET=utf8 |
```

> 业务其实并不复杂，原来的实现逻辑基本基于以下俩条SQL语句。（**以下数据基于本地测试库，添加了SQL_NO_CACHE条件避免查询语句被缓存**）

#### 根据查询条件得到数据总数（前端显示页数）

```
mysql> SELECT SQL_NO_CACHE COUNT(*) FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime;
+----------+
| count(*) |
+----------+
|   760611 |
+----------+
1 row in set (1.93 sec)
```

可以看到虽然换成了本地测试库只有76W条数据，也耗时了1.93秒，使用explain分析下SQL语句，如下

```
mysql> EXPLAIN SELECT SQL_NO_CACHE COUNT(*) FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime;
+----+-------------+-------------+------+---------------+------+---------+-------+--------+------------------------------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref   | rows   | Extra                              |
+----+-------------+-------------+------+---------------+------+---------+-------+--------+------------------------------------+
|  1 | SIMPLE      | t_operate_0 | ref  | Fcid,Fctime   | Fcid | 8       | const | 156317 | Using index condition; Using where |
+----+-------------+-------------+------+---------------+------+---------+-------+--------+------------------------------------+
1 row in set (0.00 sec)
```

#### 优化思路
通过建表语句可以看到，只是单独对Fcid、Fctime建了索引。其实针对上面的查询语句，直接对该字段建联合索引即可。<br/>
**CREATE INDEX count ON t_operate_0 (Fcid, Fctime)**

再看看查询结果
```
mysql> SELECT SQL_NO_CACHE COUNT(*) FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime;
+----------+
| COUNT(*) |
+----------+
|   760611 |
+----------+
1 row in set (0.62 sec)

mysql> EXPLAIN SELECT SQL_NO_CACHE COUNT(*) FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime;
+----+-------------+-------------+-------+-------------------+-------+---------+------+--------+--------------------------+
| id | select_type | table       | type  | possible_keys     | key   | key_len | ref  | rows   | Extra                    |
+----+-------------+-------------+-------+-------------------+-------+---------+------+--------+--------------------------+
|  1 | SIMPLE      | t_operate_0 | range | Fcid,Fctime,count | count | 12      | NULL | 156317 | Using where; Using index |
+----+-------------+-------------+-------+-------------------+-------+---------+------+--------+--------------------------+
1 row in set (0.00 sec)
```

直接命中了联合索引，查询速度从**1.93**提高到了**0.62**。那还有没有再优化的空间呢？<br/>
**我们仔细分析下业务，前端只需要得到数据总数。其实对应表里就有自增的主键字段，直接返回该字段的最新值即可**

```
mysql> SELECT SQL_NO_CACHE Fid FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime DESC LIMIT 1;
+--------+
| Fid    |
+--------+
| 762059 |
+--------+
1 row in set (0.01 sec)
```

仅0.01s就返回了结果！虽然数字和总数有些许出入，但是其实并不影响业务！我们继续看下一条语句。

#### 根据查询条件得到分页数据

```
mysql> SELECT SQL_NO_CACHE * FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime DESC LIMIT 10000, 10;
10 rows in set (6.89 sec)

mysql> EXPLAIN SELECT SQL_NO_CACHE * FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime DESC LIMIT 10000, 10;
+----+-------------+-------------+------+---------------+------+---------+-------+--------+----------------------------------------------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref   | rows   | Extra                                              |
+----+-------------+-------------+------+---------------+------+---------+-------+--------+----------------------------------------------------+
|  1 | SIMPLE      | t_operate_0 | ref  | Fcid,Fctime   | Fcid | 8       | const | 156317 | Using index condition; Using where; Using filesort |
+----+-------------+-------------+------+---------------+------+---------+-------+--------+----------------------------------------------------+
1 row in set (0.00 sec)
```

#### 优化思路
条件基本和取总数一致，只是多limit了语句，首先还是应该通过建立联合索引覆盖查询条件(上面已经建好)<br/>
由于limit语句的原因，mysql其实还是先扫描了前10000条数据，丢弃后返回了需要的10条。针对这种情况<<高性能MySQL>>中其实有提到过利用**延迟关联**的方法来提高查询效率。优化后的SQL语句如下

```
mysql> SELECT SQL_NO_CACHE a.* FROM t_operate_0 a, (SELECT Fid FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime DESC LIMIT 10000, 10) b WHERE a.Fid = b.Fid;
10 rows in set (0.02 sec)

mysql> EXPLAIN SELECT SQL_NO_CACHE a.* FROM t_operate_0 a, (SELECT Fid FROM t_operate_0 WHERE Fcid = 2111131044822780 AND Fctime >= 1505171339 AND Fctime <= 1542992406 ORDER BY Fctime DESC LIMIT 10000, 10) b WHERE a.Fid = b.Fid;
+----+-------------+-------------+--------+-------------------+---------+---------+-------+--------+--------------------------+
| id | select_type | table       | type   | possible_keys     | key     | key_len | ref   | rows   | Extra                    |
+----+-------------+-------------+--------+-------------------+---------+---------+-------+--------+--------------------------+
|  1 | PRIMARY     | <derived2>  | ALL    | NULL              | NULL    | NULL    | NULL  |  10010 | NULL                     |
|  1 | PRIMARY     | a           | eq_ref | PRIMARY           | PRIMARY | 4       | b.Fid |      1 | NULL                     |
|  2 | DERIVED     | t_operate_0 | range  | Fcid,Fctime,count | count   | 12      | NULL  | 156317 | Using where; Using index |
+----+-------------+-------------+--------+-------------------+---------+---------+-------+--------+--------------------------+
3 rows in set (0.00 sec)
```

通过建立联合索引和利用**延迟关联**调整SQL语句，查询时间直接从**6.89s**提升到了**0.02s**！

#### 关于延迟关联
>高性能MySQL
![mysql-deferred-join]({{ site.BASE_PATH }}/assets/images/mysql-deferred-join.png){:data-action="zoom"}

#### 个人理解
- 原本的语句由于查询条件是select * where ... limit，mysql首先通过where过滤出需要返回的数据，再根据limit扫描数据，丢弃数据，最后返回正确偏移的数据。
- 优化后的语句首先通过覆盖索引拿到对应的主键，虽然也需要扫描索引数据，但是扫描索引的速度是远远大于扫描数据的，然后再通过主键关联主表得到需要的数据，这样充分利用了索引大大提高了查询效率。

---

**参考**

* [https://www.cnblogs.com/Java3y/p/9969760.html](https://www.cnblogs.com/Java3y/p/9969760.html ){:target="_blank"}
* [https://blog.csdn.net/u012817635/article/details/52277490](https://blog.csdn.net/u012817635/article/details/52277490 ){:target="_blank"}
