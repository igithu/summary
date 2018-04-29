---
layout: post
title: "HDFS 文件Recover总结"
date: 2018-04-14
tags: hdfs
---

# 背景
&emsp;&emsp;HDFS的Recover机制，主要针对异常文件关闭进行recover操作，目的是保证已经写入的数据的可读性，最大程度保证数据对外服务的稳定性；HDFS提供Lease Recovery, Block Recovery和Pipeline Recovery等Recovery机制用于容错

# Recover过程
&emsp;&emsp;当文件创建或打开时，HDFS写操作创建一个pipeline，用以在DataNode间传输和存储数据
![image01](https://igithu.github.io/summary/images/recover-f1.png)

* 在客户端写hdfs文件时，其必须获取一个该文件lease，用以保证单一互斥写。如果客户端需要持续的写，其必须在写期间周期性的renew该lease。如果客户端未续借，lease将过期，此时HDFS将会关闭该文件并回收该lease，以让其它客户端能写文件，这个过程称之为Lease Recovery。
* 在写数据的过程中，如果文件的最last block没有写到pipeline的所有DataNodes中，则在Lease Recovery后，不同节点的数据将不同。在Lease Recovery 关闭文件前，需保证所有复本最后一个block有相同的长度，这个过程称为 Block Recovery。仅仅当文件最后一个block不处于COMPLETE状态时，Lease Recovery才会解决Block Recovery。
* 在pipeline写过程中，pipeline中的DataNode可能出现异常，为保证写操作不失败，HDFS需从错误中恢复并保证pipeline继续写下去。从pipeline错误中恢复的过程称为Pipeline Recovery

# Recover过程需要知道的
说明：我们把DataNode上的块Block叫做副本（Replica），以区别于NameNode上的块
## Replica状态
### FINALIZED
* Replica进入FINALIZED状态后，说明Writing已经 finished，Replica长度会被冻结（frozen），除非被re-open做append操作外
* 所有FINALIZED的Replica都有相同的generation stamp（GS），GS会在Recover时候出现递增
### RBW (Replica Being Written)
* Replica进入RBW，表明相应的文件在written或者被re-open for append中
* RBW Replica一般是打开文件最后一个block。RBW状态下对HDFS Client实际上是可读的
* 如果发生任何错误，HDFS会尝试将数据保存到RBW Replica中
* 有时候DataNode pipeline其中一台机器，突然宕机或者被重装，其他DataNode有些Replica会停留在RBW目录中
### RWR (Replica Waiting to be Recovered)
* 当DataNode宕机或者被重启时候，RBW Replica会进入RWR状态
* RWR replica会有两种结果：过期被抛弃掉，或者参与到Lease Recover过程中
### RUR (Replica Under Recovery)
* 非TEMPORARY Replica参与Lease Recover状态过程会进入RUR状态
### TEMPORARY
* 临时Replica主要用于block replication（replication monitor或者cluster balancer）
* 这个状态和RBW很相似，不过对HDFS Client是不可见的，在block replication失败时候会被删除掉

## Block状态
### UNDER_CONSTRUCTION
* UNDER_CONSTRUCTION状态表明正在写入传输中，一般处在UNDER_CONSTRUCTION的Block都是打开文件的最后Block（last block）
* UNDER_CONSTRUCTION中的Block长度和GS都并不是固定的，同时其中的数据对于Client部分可见
* NameNode上的UNDER_CONSTRUCTION Block一般用来跟踪DataNode上的RBW eplica和RWR replica
### UNDER_RECOVERY
* 当文件对应的Lease过期，此时的Last Block没有进入COMPLETE，这时需要通过Recovery过程来同步数据
* 当处在UNDER_CONSTRUCTION状态下的Last Block对应的Lease过期（lease expires），这个Block进入UN DER_RECOVERY开始Block Recovery
### COMMITTED
* COMMITTED的block，Client已经将该部分数据连同GS和Length数据写入DataNodes，但是NameNode还没有收到关于这些Replicas确认
* COMMITTED Block的数据和相应的GS不在变化 除非有append操作，这时的COMMITTED Block一般小于与DatNode上的FINALIZED replicas的最小副本数
* COMMITTED block会跟踪DataNode上的RBW replicas和 FINALIZED replicas相应的GS和长度。当Client NameNode向文件中增加新的Block或者Close文件，UNDER_CONSTRUCTION block会进入COMMITTED
* 当Last或者Second-To-Last Block在COMMIT状态中，HDFS无法关闭该文件，日志可能会反复出现COMMIT Block Recovery
### COMPLETE
* 当NameNode确认了FINALIZED replicas达到最小副本数（这个Replica已经有固定 GS/length），该Block从COMMITTED进入COMPLETE
* 只有当所有Block都进入COMPLETE，所在文件才能被Closed。
* 有时候Block会被强制进入COMPLETE，例如Client请求分配新的Block。但是前面申请的Block没有在COMPLETE

# Lease Recovery
# Block Recovery
# Pipeline Recovery



# 参考文档
* [Understanding HDFS Recovery Processes (Part 1)](http://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/)

* [Understanding HDFS Recovery Processes (Part 2)](https://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)


