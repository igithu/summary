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
&emsp;&emsp;HBase中，WAL的实现类为HLog （`在0.98中实际上是FSHLog去实现执行，以下统称为HLog`）,在写入的时候，每个Region Server拥有一个HLog日志，所有region的写入都是写到同一个HLog，写入时候RegionServer就会调用HLog.append()，将数据对<HLogKey,WALEdit>按照顺序追加到HLog中
#### 主要结构（网图）
![image01](https://igithu.github.io/summary/images/jhlog.png)

#### 主要代码段
```java
/**
   * Append a set of edits to the log. Log edits are keyed by (encoded)
   * regionName, rowname, and log-sequence-id.
   *
   * Later, if we sort by these keys, we obtain all the relevant edits for a
   * given key-range of the HRegion (TODO). Any edits that do not have a
   * matching COMPLETE_CACHEFLUSH message can be discarded.
   *
   * <p>
   * Logs cannot be restarted once closed, or once the HLog process dies. Each
   * time the HLog starts, it must create a new log. This means that other
   * systems should process the log appropriately upon each startup (and prior
   * to initializing HLog).
   *
   * synchronized prevents appends during the completion of a cache flush or for
   * the duration of a log roll.
   *
   * @param info
   * @param tableName
   * @param edits
   * @param clusterIds that have consumed the change (for replication)
   * @param now
   * @param doSync shall we sync?
   * @param sequenceId of the region.
   * @return txid of this transaction
   * @throws IOException
   */
  @SuppressWarnings("deprecation")
  private long append(HRegionInfo info, TableName tableName, WALEdit edits, List<UUID> clusterIds,
      final long now, HTableDescriptor htd, boolean doSync, boolean isInMemstore, 
      AtomicLong sequenceId, long nonceGroup, long nonce) throws IOException {
      if (edits.isEmpty()) return this.unflushedEntries.get();
      if (this.closed) {
        throw new IOException("Cannot append; log is closed");
      }
      TraceScope traceScope = Trace.startSpan("FSHlog.append");
      try {
        long txid = 0;
        synchronized (this.updateLock) {
          // get the sequence number from the passed Long. In normal flow, it is coming from the
          // region.
          long seqNum = sequenceId.incrementAndGet();
          // The 'lastSeqWritten' map holds the sequence number of the oldest
          // write for each region (i.e. the first edit added to the particular
          // memstore). . When the cache is flushed, the entry for the
          // region being flushed is removed if the sequence number of the flush
          // is greater than or equal to the value in lastSeqWritten.
          // Use encoded name.  Its shorter, guaranteed unique and a subset of
          // actual  name.
          byte [] encodedRegionName = info.getEncodedNameAsBytes();
          if (isInMemstore) this.oldestUnflushedSeqNums.putIfAbsent(encodedRegionName, seqNum);
          HLogKey logKey = makeKey(
            encodedRegionName, tableName, seqNum, now, clusterIds, nonceGroup, nonce);

          synchronized (pendingWritesLock) {
            doWrite(info, logKey, edits, htd);
            txid = this.unflushedEntries.incrementAndGet();
          }
          this.numEntries.incrementAndGet();
          this.asyncWriter.setPendingTxid(txid);

          if (htd.isDeferredLogFlush()) {
            lastUnSyncedTxid = txid;
          }
          this.latestSequenceNums.put(encodedRegionName, seqNum);
        }
        // TODO: note that only tests currently call append w/sync.
        //       Therefore, this code here is not actually used by anything.
        // Sync if catalog region, and if not then check if that table supports
        // deferred log flushing
        if (doSync &&
            (info.isMetaRegion() ||
            !htd.isDeferredLogFlush())) {
          // sync txn to file system
          this.sync(txid);
        }
        return txid;
      } finally {
        traceScope.close();
      }
    }
    
  // TODO: Remove info.  Unused.
  protected void doWrite(HRegionInfo info, HLogKey logKey, WALEdit logEdit, 
                         HTableDescriptor htd) throws IOException {
    if (!this.enabled) {
      return;
    }
    if (!this.listeners.isEmpty()) {
      for (WALActionsListener i: this.listeners) {
        i.visitLogEntryBeforeWrite(htd, logKey, logEdit);
      }
    }
    try {
      long now = EnvironmentEdgeManager.currentTimeMillis();
      // coprocessor hook:
      if (!coprocessorHost.preWALWrite(info, logKey, logEdit)) {
        if (logEdit.isReplay()) {
          // set replication scope null so that this won't be replicated
          logKey.setScopes(null);
        }
        // write to our buffer for the Hlog file.
        this.pendingWrites.add(new HLog.Entry(logKey, logEdit));
      }
      long took = EnvironmentEdgeManager.currentTimeMillis() - now;
      coprocessorHost.postWALWrite(info, logKey, logEdit);
      long len = 0;
      for (KeyValue kv : logEdit.getKeyValues()) {
        len += kv.getLength();
      }
      this.metrics.finishAppend(took, len);
    } catch (IOException e) {
      LOG.fatal("Could not append. Requesting close of hlog", e);
      requestLogRoll();
      throw e;
    }
  }

```


## HLog Roll
## HLog失效
## HLog清除


# 参考文档
* [HBase源码：HMaster启动过程](https://yq.aliyun.com/articles/25837)

