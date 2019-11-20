---
date: "2016-10-10"
title: 一道有关MySQL查询语句的面试题
description: mysql, php, 面试
slug: mysql-query
categories:
- Web
tags:
- mysql
comments: true
---

最近闲来无事，突然想起来之前在群里看到的一个有关MySQL查询语句的面试题，跟大家分享，顺便留作备忘。

原始数据如下：

> id  |    data   |    name     |       date
> :-: | :-------: | :---------: | :--------------:
> 1   |     30    |     kael    |      20160718
> 2   |     30    |     soul    |      20160720
> 3   |     66    |     kael    |      20160719
> 4   |     52    |     soul    |      20160719
> 5   |     99    |     kael    |      20160720
> 6   |     82    |     soul    |      20160718


希望通过SQL语句得到如下结果：

> name | Data_18 | Data_19 | Data_20
> :--: | :-----: | :-----: | :------:
> kael | 30      | 66      | 99
> soul | 82      | 52      | 30


太久没写原生查询语句，暂时想到的SQL语句如下：

```sql
SELECT name,
(SELECT data FROM `test` AS t1 WHERE t1.name = t.name AND date = '20160718') AS Data_18,
(SELECT data FROM `test` AS t2 WHERE t2.name = t.name AND date = '20160719') AS Data_19,
(SELECT data FROM `test` AS t3 WHERE t3.name = t.name AND date = '20160720') AS Data_20
FROM `test` AS t GROUP BY name
```

PS: 如果有更好的想法，欢迎与我交流。