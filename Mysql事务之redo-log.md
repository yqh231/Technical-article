---
title: MySQL之redo log
date: 2018-04-02 10:24:54
tags: 数据库
---
![mysql](http://opxvbng4q.bkt.clouddn.com/mysql.png)
## Redo log是什么?
MySQL数据库作为现在互联网公司内最流行的关系型数据库，相信大家都有工作中使用过。InnoDB是MySQL里最为常用的一种存储引擎，主要面向在线事务处理(OLTP)的应用。今天就让我们来探究一下InnoDB是如何一步一步实现事务的，这次我们先讲事务实现的redo log。

首先我们先明确一下InnoDB的修改数据的基本流程，当我们想要修改DB上某一行数据的时候，InnoDB是把数据从磁盘读取到内存的缓冲池上进行修改。这个时候数据在内存中被修改，与磁盘中相比就存在了差异，我们称这种有差异的数据为**脏页**。InnoDB对脏页的处理不是每次生成脏页就将脏页刷新回磁盘，这样会产生海量的IO操作，严重影响InnoDB的处理性能。对于此，InnoDB有一套完善的处理策略，与我们这次主题关系不大，表过不提。既然脏页与磁盘中的数据存在差异，那么如果在这期间DB出现故障就会造成数据的丢失。为了解决这个问题，redo log就应运而生了。
<!--more-->

## Redo log工作原理
在讲Redo log工作原理之前，先来学习一下MySQL的一些基础：

**一、日志类型**
```mermaid
graph LR
    A[MySQL日志类型]
    A-->B["物理日志(存储了数据被修改的值)"]
    A-->C["逻辑日志(存储了逻辑SQL修改语句)"]
```
redo log在数据库重启恢复的时候被使用，因为其属于物理日志的特性，恢复速度远快于逻辑日志。而我们经常使用的binlog就属于典型的逻辑日志。

**二、 checkpoint**

坦白来讲checkpoint本身是比较复杂的，checkpoint所做的事就是把脏页给刷新回磁盘。所以，当DB重启恢复时，只需要恢复checkpoint之后的数据。这样就能大大缩短恢复时间。当然checkpoint还有其他的作用。

**三. LSN(Log Sequence Number)**

LSN实际上就是InnoDB使用的一个版本标记的计数，它是一个单调递增的值。数据页和redo log都有各自的LSN。我们可以根据数据页中的LSN值和redo log中LSN的值判断需要恢复的redo log的位置和大小。

好的，现在我们来看看redo log的工作原理。说白了，redo log就是存储了数据被修改后的值。当我们提交一个事务时，InnoDB会先去把要修改的数据写入日志，然后再去修改缓冲池里面的真正数据页。

我们着重看看redo log是怎么一步步写入磁盘的。redo log本身也由两部分所构成即重做日志缓冲(redo log buffer)和重做日志文件(redo log file)。这样的设计同样也是为了调和内存与磁盘的速度差异。InnoDB写入磁盘的策略可以通过`innodb_flush_log_at_trx_commit`这个参数来控制。
```mermaid
graph LR
    A["innodb_flush_log_at_trx_commit"]
    A-->B["该值为1时表示事务提交必须调用一次fsync参数"]
    A-->C["该值为0时表示事务提交不写入磁盘，写入过程在master thread中进行"]
    A-->D["该值为2表示事务提交时不写入重做日志文件，而是写入文件系统缓冲中"]
```
当该值为1当然时最安全的，但是数据库性能为受一定影响。为0时性能较好，但是会丢失掉master thread还没刷新进磁盘部分的数据。这里我想简单介绍一下master thread，这是InnoDB一个在后台运行的主线程，从名字就能看出这个线程相当的重要。它做的主要工作包括但不限于：刷新日志缓冲，合并插入缓冲，刷新脏页等。master thread大致分为每秒运行一次的操作和每10秒运行一次的操作。master thread中刷新数据，属于checkpoint的一种。所以如果在master thread在刷新日志的间隙，DB出现故障那么将丢失掉这部分数据。当该值为2时，当DB发生故障能恢复数据。但如果操作系统也出现宕机，那么就会丢失掉，文件系统没有及时写入磁盘的数据。

这里说明一下，`innodb_flush_log_at_trx_commit`设为非0的值，并不是说不会在master thread中刷新日志了。master thread刷新日志是在不断进行的，所以redo log写入磁盘是在持续的写入。

DB宕机后重启，InnoDB会首先去查看数据页中的LSN的数值。这个值代表数据页被刷新回磁盘的LSN的大小。然后再去查看redo logLSN的大小。如果数据页中的LSN值大说明数据页领先于redo log刷新回磁盘，不需要进行恢复。反之需要从redo log中恢复数据。
