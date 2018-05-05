---
layout: post
title: "HBase HLog生命周期"
date: 2018-05-05
tags: hbase
---

&emsp;&emsp;这里主要描述HLog的整个生命周期，包括：HLog的生成，HLog Roll，HLog失效，HLog删除

# HLog构建数据
## 关键结构
&emsp;&emsp;HBase中，WAL的实现类为HLog （`在0.98中实际上是FSHLog去实现执行，以下统称为HLog`）,在写入的时候，每个Region Server拥有一个HLog日志，所有region的写入都是写到同一个HLog，写入时候RegionServer就会调用HLog.append()，将数据对<HLogKey,WALEdit>按照顺序追加到HLog中
&emsp;&emsp;其中HLogKey由sequenceid、writetime、clusterid、regionname以及tablename组成 ，这些info，通过append->doWrite->（ this.pendingWrites.add(new HLog.Entry(logKey, logEdit))） 生成。其中对于sequenceid
* 自增序号。很好理解，就是随着时间推移不断自增，不会减小。
* 一次行级事务的自增序号。行级事务是什么？简单点说，就是更新一行中的多个列族、多个列，行级事务能够保证这次更新的原子性、一致性、持久性以及设置的隔离性，HBase会为一次行级事务分配一个自增序号。
* 是region级别的自增序号。每个region都维护属于自己的sequenceid，不同region的sequenceid相互独立。
* 这里提一下，实际上后续新版1.x版本以上 HBase中已经将mvcc和sequenceid合并在一起

## 主要结构（网图）
![image01](https://igithu.github.io/summary/images/hlog.png)

# HLog Roll
&emsp;&emsp;HBase后台启动了线程LogRoller，会每隔一段时间（由参数’hbase.regionserver.logroll.period’决定）进行日志滚动，即新生成一个新的日志文件。同时HLog日志文件并不是一个大文件，而是会产生很多小文件。这样做为了能够及时删除掉“过期”已经没有用的日志数据
## Roll主要组件
&emsp;&emsp;以下点位记录<=xxxxtxid，都是已完成状态
* AsyncNotifier：异步通知组件使用，主要notify pending在Buffer上的Write Handler
  * `flushedTxid`：记录最后的flush点位，在AsyncWriter sync之后更新为lastSyncedTxid
  * `lastNotifiedTxid`：上次notify的点位，主要在flushedTxid被更新的时候，会给更新为flushedTxid
  * `flushedTxid <= lastNotifiedTxid`时， AsyncNotifier会进入wait，直到flushedTxid更新
* AsyncSyncer：主要负责向AsyncWriter发送sync请求 
  * `writtenTxid`：写入WALEdits数据点位，<=writtenTxid都已经写入
  * `lastSyncedTxid`：最后sync的点位
  * `writtenTxid <= lastSyncedTxid`时AsyncSyncer进入wait状态
* AsyncWriter：主要负责flush本地Buffer中的数据，将WALEdit数据持久化到HDFS上
  * `pendingTxid`：Write Handler pending的点位
  * `lastWrittenTxid`：记录上一次写入的点位
  * `pendingTxid <= lastWrittenTxid`时，AsyncWriter会进入wait状态，直到writtenTxid被更新
* LogRoller：负责周期HLog Roller
  * 驱动数据落盘
  * Roll新HLog，等待下一轮append + sync
* SequenceFileLogWriter：主要负责delegates to SequenceFile.Writer
* 以上点位(txid)之间关系
  ![image02](https://igithu.github.io/summary/images/txid.png)


## Roll全局关键过程

![image03](https://igithu.github.io/summary/images/hlog-disk.png)
注意两点
* 一次doWrite，生成一次txid，生成一次sequenceId；txid与sequenceId没有必然联系，有时候可以关联起来
* 一次rollWrite，roll一次新文件：除了LogRoller周期roll文件外，在向pendingWrites add数据或者调用append数据到HDFS过程中出现异常，一般都会输出Fatal日志，然后调用requestLogRoll来触发rollWrite

# HLog失效
&emsp;&emsp;数据从Memstore中落盘，对应的日志就可以被删除，因此一个文件所有数据失效，只要看该文件中最大sequenceid对应的数据是否已经落盘就可以，HBase会在每次执行flush的时候纪录对应的最大的sequenceid，如果前者小于后者，则可以认为该日志文件失效。一旦判断失效就会将该文件从.logs目录移动到.oldlogs目录,


# HLog清除
&emsp;&emsp;HMaster后台会启动一个线程LogCleaner,每隔一段时间（由参数’hbase.master.cleaner.interval’，默认1分钟）会检查一次文件夹.oldlogs下的所有失效日志文件，确认是否可以被删除，确认之后执行删除操作。又有同学问了，刚才不是已经确认可以被删除了吗？这里基于两点考虑，第一对于使用HLog进行主从复制的业务来说，第三步的确认并不完整，需要继续确认是否该HLog还在应用于主从复制；第二对于没有执行主从复制的业务来讲，HBase依然提供了一个过期的TTL（由参数’hbase.master.logcleaner.ttl’决定，默认10分钟），也就是说.oldlogs里面的文件最多依然再保存10分钟。

