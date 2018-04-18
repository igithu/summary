---
layout: post
title: "JVM内存回收总结"
date: 2018-04-15
tags: jvm，hbase
---


# JVM堆内
## 各个区域与参数的配置对应关系
![image01](https://igithu.github.io/summary/images/jvm-con.png)
![image02](https://igithu.github.io/summary/images/gc-ratio.png)
  
  
## Survivor区为什么存在
&emsp;&emsp;Survivor区实际上是为了增长对象在Young区存在的周期，让对象回收变得可控。假设没有Survivor，eden区每进行一次Minor GC，存活的对象就会被送到Old区，Old区域空间很快耗尽，触发Full GC，频繁触发Full GC明显会影响系统的性能（频发的Full GC消耗的时间是非常可观的，这一点会影响大型程序的执行和响应速度，更不要说某些连接会因为超时发生连接错误了）

  

## 为什么存在两个Survivor
### 误解
一种说法是为了避免Young区内存空间的碎片化，S0、S1回收后的每次交换整理会减少碎片化。但是这种说法貌似是站不住脚本：
* Minor GC: 使用的回收算法为复制收集算法
* Full GC: 一般采用的是标记—整理算法
Minor gc比较频繁、对象存活率低，用复制算法在回收时的效率会更高，也不会产生内存碎片，所以以上理由应该是不对的

### 一种理解
&emsp;&emsp;如果是一个Survivor，实际就没有对象在Young区域循环存在（s0，s1之间），对象在Yong区域存在的周期也就不太可控，我理解应该是利用两个S区域，让对象在S区域存在变得可控（如果对象的复制次数达到16次，该对象就会被送到Old区域中）。所以两个S区域是有必要的

# HBase下的主要JVM变量
## -Xms16g
* 初始进程内的堆内存大小
## -Xmx16g
* 最大分配堆内存大小
## -Xmn2g
* 用来设置堆内新生代的大小。通过这个值我们也可以得到老生代的大小：-Xmx减去-Xmn
## -XX:SurvivorRatio=2
* 设置年轻代中Eden区与Survivor区的大小比值，具体见上图
## -XX:CMSInitiatingOccupancyFraction=70
* 参数来设置老年代空间使用百分比,达到百分比就进行垃圾回收。
* 这个参数默认是92%，参数选择需要看具体的应用场景。
* 设置的太小会导致频繁的CMS GC，产生大量的停顿
## -XX:+UseCMSInitiatingOccupancyOnly
* 标志来命令JVM不基于运行时收集的数据来启动CMS垃圾收集周期。而是，当该标志被开启时，JVM通过CMSInitiatingOccupancyFraction的值进行每一次CMS收集，而不仅仅是第一次。然而，请记住大多数情况下，JVM比我们自己能作出更好的垃圾收集决策。因此，只有当我们充足的理由(比如测试)并且对应用程序产生的对象的生命周期有深刻的认知时，才应该使用该标志。
* 如果没有设置-XX:+UseCMSInitiatingOccupancyOnly，虚拟机会根据收集的数据决定是否触发（建议线上环境带上这个参数，不然会加大问题排查的难度）
## -XX:+UseConcMarkSweepGC
* 默认不启用 启用CMS低停顿垃圾收集器
## -XX:-OmitStackTraceInFastThrow
* 强制要求JVM始终抛出含堆栈的异常
## -XX:+CMSScavengeBeforeRemark
* 设置后，在执行CMS remark之前进行一次youngGC，这样能有效降低remark的时间，如果没有加这个参数，remark时间最大能达到3s，加上这个参数之后remark时间减少到1s之内。
## -XX:+UnlockDiagnosticVMOptions
* 解锁诊断JVM选项的选项
## -XX:ParallelGCThreads=24
* 为了提高垃圾回收的性能，java在parallel回收的时候可以设置同时并行处理的线程数也就是ParallelGCThreads，如果你没有设置该参数，该参数jvm会默认设置成online的cpu的核数但并不包括被shutdown的cpu的核数
## -XX:+PrintPromotionFailure
* Promotion Failed时候对象晋升的情况
## -XX:+PrintGC
* 开启了简单GC日志模式，为每一次新生代（young generation）的GC和每一次的Full GC打印一行信息
## -XX:+PrintGCDetails
* 打印GC详细信息
## -XX:+PrintGCDateStamps
* 将时间和日期也加到GC日志中
## -XX:+UseGCLogFileRotation
* 启用GC日志文件的自动转储
## -XX:NumberOfGCLogFiles=5
* GC日志文件的循环数目 
## -XX:GCLogFileSize=200M
* 控制GC日志文件的大小
## -XX:PermSize=256m
## -XX:+BlockOffsetArrayUseUnallocatedBlock


  


# 参考文档

* [JVM内存管理------GC算法精解（复制算法与标记/整理算法）](https://www.cnblogs.com/zuoxiaolong/p/jvm5.html)
