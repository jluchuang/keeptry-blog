---
title: 关于Innodb的一些名词
date: 2017-03-10 
tags: [mysql, innodb]
toc: true
category: mysql
description: 从一个初学者的角度来看, Mysql能够称为当前最流行的关系型数据库之一, 与Mysql底层的插件式存储引擎支持是分不开的, 而其中最流行的两个存储引擎InnoDB和MyISAM, MyISAM主要是面向OLAP服务, 而InnoDB, 完整的支持数据库的ACID特性, 能够提供良好的OLTP服务. 所以这里简记以下关于InnoDB的ACID我们需要了解的一些名词和概念. 这里只是概念,没有深度的分析.
---

## 写在前面
从一个初学者的角度来看, Mysql能够成为当前最流行的关系型数据库之一, 与Mysql底层的插件式存储引擎支持是分不开的, 而其中最流行的两个存储引擎InnoDB和MyISAM, MyISAM主要是面向OLAP服务, 而InnoDB, 完整的支持数据库的ACID特性, 能够提供良好的OLTP服务. 所以这里简记以下关于InnoDB的ACID我们需要了解的一些名词和概念. 这里只是概念,没有深度的分析.

## OLAP 与 OLTP
- OLTP(Online Transaction processing) is characterized by a large number of short on-line transactions (INSERT, UPDATE, DELETE). The main emphasis for OLTP systems is put on very fast query processing, maintaining data integrity in multi-access environments and an effectiveness measured by number of transactions per second. In OLTP database there is detailed and current data, and schema used to store transactional databases is the entity model (usually 3NF).

- OLAP(Online Analytical processing) is characterized by relatively low volume of transactions. Queries are often very complex and involve aggregations. For OLAP systems a response time is an effectiveness measure. OLAP applications are widely used by Data Mining techniques. In OLAP database there is aggregated, historical data, stored in multi-dimensional schemas (usually star schema).

## InnoDB的索引
- B+树: 主流数据库之所以采用B+树作为索引的数据结构, 与当前操作系统的硬盘寻址有关.
- 索引表: InnoDB中每个数据表的结构组织都是一个B+树
- 聚簇索引(cluster index): 即按照每张表的主键构造一颗B+树, 同时叶子节点中存放的即为整张表的行为记录数据, 聚簇索引的叶子节点即为数据页.
- 辅助索引(secondary index): 叶子节点并不包含行记录的全部数据, 叶子节点除了键值意外, 还包含了InnoDB的聚簇索引键.
- 关于索引的一些优化: ICP(Index Column Pushdown), Covering Index, MRR(Multi-Range Read).
- 其实索引的使用往往与我们的查询语句的where查询条件直接相关, 这里强烈推荐[何登成:SQL中的where条件,在数据库中提取与应用浅](http://hedengcheng.com/?p=577)

## InnoDB的锁 (I, Isolation)
- MVCC: Multiversion concurrency control
- 两阶段锁
- 锁的对象:  InnoDB的锁可以细化到具体的行记录(记录锁).以及记录在索引之间的间隙(间隙锁)
- S锁/X锁/IS锁/IX锁/GAP锁 对于Mysql所得分析还是推荐一波:[何登成:MySQL 加锁处理分析](http://hedengcheng.com/?p=771)

## InnoDB的事务与Undo/Redo/Purge (A, C, D)
- 事务分类
- redo: Redo主要用来实现数据的持久性, 包括两部分重做日志缓冲(redo log buffer), 其是易失的; 重做日志文件(redo log file), 其是持久的. InnoDB的是事务的存储引擎, 通过Force Log at Commit 机制实现事务的持久性, 即当事务提交(commit)时必须现将该事务的所有日志写入到重做日志文件进行持久化, 待事务的commit操作完成才算完成.这里的重做日志文件由两部分组成, 在InnoDB存储引擎中, redo log用来保证事务的持久性, undo log用来帮助事务回滚以及MVCC的功能. redo log基本上是顺序读写的, 在数据库运行时不需要对redo log的文件进行读取操作. 而undo log是需要随机读写的.
- undo: Undo存放在数据库内部的一个特殊端中, 这个段称为undo段(undo segment), undo 段位于共享表空间内.undo 存储的知识逻辑日志, 因此在事务回滚的时候, 其实知识将数据库逻辑的会滚到操作之前的样子,但是数据结构以及数据页本身可能在回滚之后大不相同.因为在多用户并发系统中, 可能会有多个并发的事务, 不能因为一个事务的回滚而影响到其他事务.
- purge: Purge用于最终完成delete和update操作.这样设计是因为InnoDB存储引擎支持MVCC,所以记录不能在事务提交时立即进行处理.这是其他事务可能正在引用这一行, 故InnoDB需要保存记录之前的版本, 而是否删除该条记录通过purge来进行判断, 若判断该行记录已经不被任何其他事务引用,那么就可以进行真正的delete操作.
- 最后推荐:[何登成:InnoDB 事务/锁/多版本 实现分析](http://hedengcheng.com/?p=286)

## 分布式事务
- XA事务(2pc)


