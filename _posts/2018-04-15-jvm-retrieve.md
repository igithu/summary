---
layout: post
title: "JVM内存回收总结"
date: 2018-04-15
tags: jvm
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




# 参考文档

* [JVM内存管理------GC算法精解（复制算法与标记/整理算法）](https://www.cnblogs.com/zuoxiaolong/p/jvm5.html)
