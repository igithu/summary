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
&emsp;&emsp;其中HLogKey由sequenceid、writetime、clusterid、regionname以及tablename组成 ，这些info，通过append->doWrite->（ this.pendingWrites.add(new HLog.Entry(logKey, logEdit))） 生成。其中对于sequenceid
* 自增序号。很好理解，就是随着时间推移不断自增，不会减小。
* 一次行级事务的自增序号。行级事务是什么？简单点说，就是更新一行中的多个列族、多个列，行级事务能够保证这次更新的原子性、一致性、持久性以及设置的隔离性，HBase会为一次行级事务分配一个自增序号。
* 是region级别的自增序号。每个region都维护属于自己的sequenceid，不同region的sequenceid相互独立。
* 这里提一下，实际上后续新版1.x版本以上 HBase中已经将mvcc和sequenceid合并在一起

#### 主要结构（网图）
![image01](https://igithu.github.io/summary/images/hlog.png)

#### 主要代码段（HBase版本：0.98.8）
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
&emsp;&emsp;HBase后台启动了线程LogRoller，会每隔一段时间（由参数’hbase.regionserver.logroll.period’决定）进行日志滚动，即新生成一个新的日志文件。同时HLog日志文件并不是一个大文件，而是会产生很多小文件。这样做为了能够及时删除掉“过期”已经没有用的日志数据
### Roll主要组件
&emsp;&emsp;以下点位记录都是<=xxxxtxid都是已完成操作
* AsyncNotifier：异步通知组件使用，主要notify pending在Buffer上的Write Handler
  * `flushedTxid`：记录最后的flush点位，在AsyncWriter sync之后更新为lastSyncedTxid
  * `lastNotifiedTxid`：上次notify的点位，主要在flushedTxid被更新的时候，会给更新为flushedTxid
  * `flushedTxid <= this.lastNotifiedTxid`时， AsyncNotifier会进入wait，直到flushedTxid更新
* AsyncSyncer：主要负责向AsyncWriter发送sync请求 
  * `writtenTxid`：写入WALEdits数据点位，<=writtenTxid都已经写入
  * `lastSyncedTxid`：最后sync的点位
  * `writtenTxid <= lastSyncedTxid`时AsyncSyncer进入wait状态
* AsyncWriter：主要负责flush本地Buffer中的数据，将WALEdit数据持久化到HDFS上
  * `pendingTxid`：Write Handler pending的点位
  * `lastWrittenTxid`：记录上一次写入的点位
  * `pendingTxid <= lastWrittenTxid`时，AsyncWriter会进入wait状态，直到writtenTxid被更新
### Roll关键过程
* Write Handler将日志数据向HLog写入pending buffer中，之后notify到AsyncWriter线程：有新的WALEdit是数据在local buffer中
* Write Handler接下来会等待下游线程HLog.sync()完成同步（以txid为单位）
* AsyncWriter 线程会在后台收集在pending buffer中的WALEdit Log数据，flush只写数据到HDFS上，notify AsyncSyncer在pending buffer一部分已经flush到HDFS上，可以进行下一步的sync


### Roll相关代码


## HLog失效
## HLog清除


# 参考文档
* [HBase源码：HMaster启动过程](https://yq.aliyun.com/articles/25837)

