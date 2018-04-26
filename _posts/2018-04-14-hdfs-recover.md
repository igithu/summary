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

* 在客户端写hdfs文件时，其必须获取一个该文件lease，用以保证单一写。如果客户端需要持续的写，其必须在写期间周期性的续借(renew)该lease。如果客户端未续借，lease将过期，此时HDFS将会关闭该文件并回收该lease，以让其它客户端能写文件，这个过程称之为Lease Recovery。
* 在写数据的过程中，如果文件的最后一个block没有写到pipeline的所有DataNodes中，则在Lease Recovery后，不同节点的数据将不同。在Lease Recovery 关闭文件前，需保证所有复本最后一个block有相同的长度，这个过程称为 Block Recovery。仅仅当文件最后一个block不处于COMPLETE状态时，Lease Recovery才会解决Block Recovery。
* 在pipeline写过程中，pipeline中的DataNode可能出现异常，为保证写操作不失败，HDFS需从错误中恢复并保证pipeline继续写下去。从pipeline错误中恢复的过程称为Pipeline Recovery

# Lease Recovery
# Block Recovery
# Pipeline Recovery



# 参考文档
* [Understanding HDFS Recovery Processes (Part 1)](http://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/)

* [Understanding HDFS Recovery Processes (Part 2)](https://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)


