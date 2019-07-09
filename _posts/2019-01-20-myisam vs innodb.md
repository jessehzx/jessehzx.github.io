---
layout:     post
title:      为什么你要用 InnoDB, 而不是 MyISAM ?
subtitle:   简单捋一捋 MySQL 的两种存储引擎 ...
date:       2019-01-20            
author:     jessehzx                
header-img: img/pexels-photo-1654495.jpeg
catalog: 	  true
tags:
    - MyISAM
    - InnoDB
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！


#### 引子

团队里有位成员接了个需求：按天统计出操作日志。由于系统运行有一段时间了，目前涉及的 MySQL 数据库表数据量已达到上亿，他担心写好代码后执行 SQL 速度过慢影响性能。我们看了下这个表用的存储引擎是 InnoDB，对于这种查询比较多的需求，如果用 MyISAM 引擎是否就好呢？

#### MyISAM 与 InnoDB

众所周知，MySQL 有两种常见的存储引擎。一种是 MyISAM，一种是 InnoDB。

> 我发现很多人在交流技术的时候，对英文技术名词的读法千奇百怪，甚至借助肢体语言、丰富的表情，试图表意，双方才能频频点头（潜台词：搞了半天原来你说的是这个东西啊~），所以要养成一种习惯：对于还有些模糊的技术名词，都要引发自己的兴趣去考究一番。

那，贴一下这两个存储引擎的单词发音吧，试着读几遍你就记住了。

| 技术名词 | 发音 |
|---|---|
| MyISAM | [maɪ-zeim] |
| InnoDB | [,ɪnnə-db] |

接下来我们尝试回答几个问题。

***一、它们是什么？***

先来看看官网对 [MyISAM](https://dev.mysql.com/doc/refman/8.0/en/myisam-storage-engine.html) 的描述，只有一句话，看来官方也不想多加解释。

> MyISAM is based on the older (and no longer available) ISAM storage engine but has many useful extensions.

大意：MyISAM 是一款青出于蓝而胜于蓝的存储引擎，它在 ISAM 基础上作了一些扩展和加工。关于 ISAM ，我只告诉你它是 Indexed Sequential Access Method 的缩写，翻译为“有索引的顺序访问方法”。

而对 [InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html) 的描述，就更 professional 一些了。

> InnoDB is a general-purpose storage engine that balances high reliability and high performance. In MySQL 8.0, InnoDB is the default MySQL storage engine. Unless you have configured a different default storage engine, issuing a CREATE TABLE statement without an ENGINE= clause creates an InnoDB table.

大意：InnoDB 是一种通用的存储引擎，在高可靠和高性能上作了均衡。MySQL 8.0 中，它是默认的存储引擎（其实在5.5之后的版本就是了），当你执行 CREATE TABLE 建表语句并且不带 “ENGINE = ”子句时，默认帮你创建的就是 InnoDB 表了。

***二、两者有什么区别？***

每个引擎都有利有弊。

先还是拿官网两者的 Features 来作一个分析对比吧：

![](https://ws3.sinaimg.cn/large/006tNc79gy1fzcstmyczkj31bu0u07e7.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1fzcsnf4fb1j317n0u0wpl.jpg)

1、InnoDB 是聚集索引，数据文件是和索引绑在一起的，必须要有主键，通过主键索引效率很高，但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，否则其他索引也会很大。而 MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针，主键索引和辅助索引是独立的。

2、InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败。

3、InnoDB 在 MySQL 5.6 之前不支持全文索引，而 MyISAM 一直都支持，如果你用的是老版本，查询效率上 MyISAM 要高。  

4、InnoDB 锁粒度是行锁，而 MyISAM 是表锁。

5、InnoDB 支持事务，MyISAM 不支持，对于 InnoDB 每一条 SQL 语言都默认封装成事务，自动提交，这样会影响速度，所以最好把多条 SQL 语言放在 begin 和 commit 之间，组成一个事务。

6、InnoDB 不保存表的具体行数，执行 select count(*) from table 时需要全表扫描。而 MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快，但如果上述语句还包含了 where 子句，那么两者执行效率是一样的。

***三、说了这么多，我到底该选哪个？***

两种存储引擎的选择，要结合你的业务场景来做选型，可以参考以下基本原则：

1、是否要支持事务，如果要请选择 Innodb，如果不需要可以考虑 MyISAM。

2、如果表中绝大多数都是读查询（有人总结出 读:写比率大于100:1），可以考虑 MyISAM，如果既有读又有写，而且也挺频繁，请使用 InnoDB。

3、系统崩溃后，MyISAM 恢复起来更困难，能否接受。

4、MySQL 5.5 开始 InnoDB 已经成为 MySQL 的默认引擎(之前是 MyISAM )，说明其优势是有目共睹的，如果你不知道用什么，那就用InnoDB吧，至少不会差。

#### 后记

那我们试图来回答刚开始提到的问题：是否当初设计表结构的时候就要用 MyISAM 呢？不见得，试想一下，既然是一张用来记录操作人员的行为表，那肯定涉及到大量的写操作，多人使用还涉及批量操作，逻辑复杂一点还要用到事务控制。后面的需求虽然全是读需求，但如果索引设计合理，SQL 语句写得够好，性能一样没什么问题。

<div align=center>(完）

<div align=center><img src="https://user-gold-cdn.xitu.io/2018/11/16/1671a288a68d53b7?w=440&h=453&f=jpeg&s=37712" width=30% height=30%/>


