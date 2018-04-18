---
layout: post
title: "HBase RegionServer宕机恢复总结"
date: 2018-04-14
tags: hbase
---

# 背景
&emsp;&emsp;HBase宕机恢复是HBase一个重要部分。在HBase实现中，并没有直接写入数据文件到磁盘上，而是先写入MemStore上，然后在一定条件下将MemStore刷到磁盘上，但是仅有这个实现会存在RegionServer宕机丢数据的情况；所以避免宕机丢数据，RegionServer进程写入MemStore之前都会执行顺序写入HLog操作到文件系统的磁盘上，如果RegionServer进程crash或者机器宕机，HBase会根据已经写入的HLog进行数据恢复处理

# HLog的生命周期
&emsp;&emsp;宕机恢复实际上是依靠HLog进行的 所以HLog的声明周期:Roll，失效，以至于最后被删除掉是很重要的 这里首先梳理HLog在正常情况的生命周期，

## HLog构建数据
&emsp;&emsp;HBase中，WAL的实现类为HLog,在写入的时候，每个Region Server拥有一个HLog日志，所有region的写入都是写到同一个HLog，写入时候RegionServer就会调用HLog.append()，将数据对<HLogKey,WALEdit>按照顺序追加到HLog中
#### 主要结构（网图）
![image01](https://igithu.github.io/summary/images/jhlog.png)

#### 主要代码段
```java
/**
   * @param info
   * @param tableName
   * @param edits
   * @param clusterId The originating clusterId for this edit (for replication)
   * @param now
   * @param doSync shall we sync?
   * @param replayed edit is replayed or not
   * @return txid of this transaction
   * @throws IOException
   */
  private long append(HRegionInfo info, byte [] tableName, WALEdit edits, UUID clusterId,
      final long now, HTableDescriptor htd, boolean doSync, boolean replayed)
    throws IOException {
      if (edits.isEmpty()) return -1;
      if (this.closed) {
        throw new IOException("Cannot append; log is closed");
      }
      long txid = 0;
      long start = System.currentTimeMillis();
      byte[] encodedRegionName = info.getEncodedNameAsBytes();
      HashedBytes encodedRegionNameHashBytes = new HashedBytes(encodedRegionName);
      edits.setReplayed(replayed);
      
      EntrySlot slot = asyncProcessor.nextSlot();
      try {
        txid = slot.getTxid();
        long seqNum = obtainSeqNum(txid);
        this.lastSeqWritten.putIfAbsent(encodedRegionNameHashBytes, seqNum);
        HLogKey logKey = makeKey(encodedRegionName, tableName, seqNum, now, clusterId);
        doWrite(slot, info, logKey, edits, htd);
      } finally {
        slot.publish();
      }

      this.numEntries.incrementAndGet();
      if (htd.isDeferredLogFlush()) {
        lastDeferredTxid = txid;
      }
      // Sync if catalog region, and if not then check if that table supports
      // deferred log flushing
      if (doSync && 
          (info.isMetaRegion() ||
          !htd.isDeferredLogFlush())) {
        // sync txn to file system
        this.sync(txid);
      }
      long took = System.currentTimeMillis() - start;
      doWALTime.inc(took);
      return txid;
    }
```



## HLog Roll
## HLog失效
## HLog清除


# 参考文档
* [HBase源码：HMaster启动过程](https://yq.aliyun.com/articles/25837)

