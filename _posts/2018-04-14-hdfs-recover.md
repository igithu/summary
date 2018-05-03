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

# Recover过程中的重要元素
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
### 转换关系
![image02](https://igithu.github.io/summary/images/rep-state.png)

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
### 转换关系
![image03](https://igithu.github.io/summary/images/block-state.png)

## HDFS的Lease和LeaseManager
### Lease
&emsp;&emsp;Lease是为了实现一个文件在一个时刻只能被一个客户端写。Client写文件时需要先申请一个Lease，一旦有客户端持有了某个文件的Lease，其它Client就不可能再申请到该文件的Lease，这就保证了同一时刻对一个文件的写操作只能发生在一个客户端。Client写数据的过程中，后台线程会续租（renew lease）
&emsp;&emsp;NameNode的LeaseManager是Lease机制的核心，维护了文件与Lease、客户端与Lease的对应关系，这些信息会随写入数据的变化实时发生对应改变
### LeaseManager
* 核心数据结构
  * leases：记录Client（leaseHolder）与Lease的映射关系
  * sortedLeases：Lease集合
  * sortedLeasesByPath：文件路径到Lease的映射关系
* Limit超时控制
  * softLimit：默认时间, 60s.
    * softLimit到期之前，一个Client拥有对他的文件的独立访问权，其他Client不能剥夺该客户端独占写这个文件的权利
    * softLimit到期之后，其他任何一个Client都可以回收lease，继而得到这个文件的lease，获得对这个文件的独占访问权
  * hardLimit：默认时间, 1hour. 到期之后，NameNode会对当前Lease进行强制回收
  * 所有回收Lease入口全部通过FSNamesystem.internalReleaseLease实现


# HDFS Recover

## Recover触发
&emsp;&emsp;当Client（典型的主要是HBase RegioServer）突然宕机或者与HDFS中断的连接，在softLimit过期之前，其他Client无法写入数据，期间无论写入或者调用recoverLease操作都会有softLimit判断；softLimit过期之后，会检查当前文件是否被close掉（实际通过INodeFile.isUnderConstruction()判断），如果没有则进入recover流程，其中Lease Recovery和Block Recovery主要目的是使文件的Last block的所有Replica数据达到一致.
&emsp;&emsp;实际Lease Recovery过程包含Block Recovery，Lease Recovery是整个过程的驱动者，Block Recovery是DataNode上的Block Recovery执行过程；

## Lease Recovery/Block Recovery
当其他客户端试图获取当前文件的Lease时候，就会进入Lease Recovery；入口：FSNamesystem.internalReleaseLease，整体运行过程分为：
* Lease Recovery预处理：NameNode执行
* Block Recovery进行实际的Recover：NameNode，DataNode执行
* Lease Recovery后处理更新Block Info：NameNode执行

### Lease Recovery预处理
* NameNode找到"Last Block"所在的DataNode，然后将其作为Primary DataNode，该Primary DataNode作为主导DataNode存在协调进行Block Recovery 
* 将当前Block更新到Primary DataNode的Description下的recoverBlocks中，之后handleHeartbeat会捕捉到进一步处理
* 其他过程
  * 文件所有的Block都Complete直接关闭文件即可，但是有Block小于配置最小副本数，会抛出异常AlreadyBeingCreatedException
  * 如果对应文件的Block在现有的DataNode上都不存在，则直接remove Block然后进行文件关闭操作
  * BlockRecover准备，获取blockRecoveryId（实际为GS），更新Leaseholder（reassignLease， renewLease）
* 关键代码
```java
public void initializeBlockRecovery(BlockInfo blockInfo, long recoveryId,
      boolean startRecovery) {
    setBlockUCState(BlockUCState.UNDER_RECOVERY);
    blockRecoveryId = recoveryId;
    if (!startRecovery) {
      return;
    }
    if (replicas.length == 0) {
      NameNode.blockStateChangeLog.warn("BLOCK*" +
          " BlockUnderConstructionFeature.initializeBlockRecovery:" +
          " No blocks found, lease removed.");
      // sets primary node index and return.
      primaryNodeIndex = -1;
      return;
    }
    boolean allLiveReplicasTriedAsPrimary = true;
    for (ReplicaUnderConstruction replica : replicas) {
      // Check if all replicas have been tried or not.
      if (replica.isAlive()) {
        allLiveReplicasTriedAsPrimary = allLiveReplicasTriedAsPrimary
            && replica.getChosenAsPrimary();
      }
    }
    if (allLiveReplicasTriedAsPrimary) {
      // Just set all the replicas to be chosen whether they are alive or not.
      for (ReplicaUnderConstruction replica : replicas) {
        replica.setChosenAsPrimary(false);
      }
    }
    long mostRecentLastUpdate = 0;
    ReplicaUnderConstruction primary = null;
    primaryNodeIndex = -1;
    for (int i = 0; i < replicas.length; i++) {
      // Skip alive replicas which have been chosen for recovery.
      if (!(replicas[i].isAlive() && !replicas[i].getChosenAsPrimary())) {
        continue;
      }
      final ReplicaUnderConstruction ruc = replicas[i];
      final long lastUpdate = ruc.getExpectedStorageLocation()
          .getDatanodeDescriptor().getLastUpdateMonotonic();
      if (lastUpdate > mostRecentLastUpdate) {
        primaryNodeIndex = i;
        primary = ruc;
        mostRecentLastUpdate = lastUpdate;
      }
    }
    if (primary != null) {
      primary.getExpectedStorageLocation().getDatanodeDescriptor()
          .addBlockToBeRecovered(blockInfo);
      primary.setChosenAsPrimary(true);
      NameNode.blockStateChangeLog.debug(
          "BLOCK* {} recovery started, primary={}", this, primary);
    }
 }
```

### Block Recovery执行
&emsp;&emsp;Block Recovery被LeaseRecovery所触发，其中recoverId实际上是GS，Block Recovery过程主要跨越NameNode和DataNode进行执行
#### NameNode执行部分
* NameNode在执行handleHeartbeat，过程中会捕捉到有需要Recovery的Block
* 执行truncate相关逻辑，过滤stale node：主要30s没有心跳的DataNode
* 发送BlockRecoveryCommand，RecoverBlock到Primary DataNode进一步进行Recover
#### DataNode执行部分
在DataNode上执行Recover的载体主要有BlockRecoveryWorker和RecoveryTaskContiguous
* Primary DataNode会接收到NameNode发送来的BlockRecoveryCommand，开始继续Recover；
* 从Block所在的DataNode上获取ReplicaRecoveryInfo，包括：GS，Length等。同时
* 过滤出含有合法的GS，length以及ReplicaState的Replica，放在BlockRecord List中
* Primary DataNode通过RPC调用DataNode Call
  * 将GS以及length更新到DataNode List中（RecoveringBlock.getLocations()）
  * 各个DataNode上Finalize（FsVolumeImpl.addFinalizedBlock）
  * 各个DataNode最后通过IBR（Incremental Block Reports）将最后的Block信息通知到NameNode
* Primary DataNode将new GS，new Length更新到NameNode中

### Lease Recovery确认Recover
* NameNode更新Block Info：new GS，new Length
* NameNode commit文件的Last Block，将Last Block标记complete（commitOrCompleteLastBlock）
* NameNode最后移除Lease，close文件，放开权限，其他Client可以进行读写操作（finalizeINodeFileUnderConstruction）
* NameNode提交这些改变到edit log

## Lease Recovery/Block Recovery整体过程
![image04](https://igithu.github.io/summary/images/recover-process.png)



## Pipeline Recovery



# 参考文档
* [Understanding HDFS Recovery Processes (Part 1)](http://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/)

* [Understanding HDFS Recovery Processes (Part 2)](https://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)


