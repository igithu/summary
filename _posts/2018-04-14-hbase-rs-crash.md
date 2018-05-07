---
layout: post
title: "HBase RegionServer宕机恢复的前世今生"
date: 2018-04-14
tags: hbase
---

# 背景
&emsp;&emsp;HBase宕机恢复是HBase一个重要部分。在HBase实现中，并没有直接写入数据文件到磁盘上，而是先写入MemStore上，然后在一定条件下将MemStore刷到磁盘上，但是仅有这个实现会存在RegionServer宕机丢数据的情况；所以避免宕机丢数据，RegionServer进程写入MemStore之前都会执行顺序写入HLog操作到文件系统的磁盘上，如果RegionServer进程crash或者机器宕机，HBase会根据已经写入的HLog进行数据恢复处理

# HLog的生命周期
&emsp;&emsp;宕机恢复实际上是依靠HLog进行的 所以HLog的声明周期:Roll，失效，以至于最后被删除掉是很重要的；关于HLog的周期见：[HBase HLog生命周期](https://igithu.github.io/summary/2018/05/hbase-hlog)


# 参考文档
* [HBase源码：HMaster启动过程](https://yq.aliyun.com/articles/25837)

* [HBase原理－RegionServer宕机数据恢复](http://hbasefly.com/2016/10/29/hbase-regionserver-recovering/)


