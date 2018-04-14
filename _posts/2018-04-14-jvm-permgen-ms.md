---
layout: post
title: "JDK8-9中的变化: PermGen VS MetaSpace总结"
date: 2018-04-14
tags: jvm
---



# 背景
这里简单总结下 Java8内存结构区别之一：MetaSpace


# Java7 PermGen

## JVM内存
* 新生代：Eden，以及s0，s1
* 老年代：Tenured
* 永久代：PermGen
&emsp;&emsp;内存回收模型这里不再描述 有兴趣的可以看下CMS和G1GC对比，来了解内存回收的过程
  
 ![image01](https://igithu.github.io/summary/images/jvm-ess-old.jpg)
    
      
      

## PermGen
## 概念
* PermGen Space的全称是 `Permanent Generation space`, 是指内存的永久保存区.
* 这一部分用于存放Class和Meta的信息，Class在被 Load的时候被放入PermGen Space区域，Class和存放Instance的Heap区域不同， GC不会在主程序运行期对PermGen Space进行清理
* GC(Garbage Collection)不会在主程序运行期对PermGen space进行清理，所以如果APP会LOAD很多CLASS 的话,就很可能出现PermGen space错误。PermGen中对象可回收的条件是，ClassLoader可以被回收 其下的所有加载过的没有对应实例的类信息（保存在PermGen）可被回收

## 伴随的问题
&emsp;&emsp;MaxPermSize设置过小的时候会出现“java.lang.OutOfMemoryError: PermGen space”，很多时候是因为类加载相关的内存泄露，新Class加载器创建导致.
&emsp;&emsp;这里的内存泄漏，是指java类和类加载器在被取消部署后不能被垃圾回收。举例：以一个部署到应用程序服务器的Java web程序来说，当该应用程序被卸载的时候,你的EAR/WAR包中的所有类都将变得无用。只要应用程序服务器还活着,JVM将继续运行,但是一大堆的类定义将不再使用,理应将它们从永久代(PermGen)中移除。如果不移除的话,我们在永久代(PermGen)区域就会有内存泄漏。这里jmap dump出内存会排查出相关的问题，在JDK7 也开业使用jmap -permstat <pid> 排查



# Java8 MetaSpace
## 概念
&emsp;&emsp;Perm这一整块内存来存klass等信息，必不可少地会配置-XX:PermSize以及-XX:MaxPermSize来控制这块内存的大小，JVM在启动的时候会根据这些配置来分配一块连续的内存块，但是随着动态类加载的情况越来越多，这块内存变得不太可控，到底设置多大合适是每个开发者要考虑的问题，如果设置太小了，系统运行过程中就容易出现内存溢出，设置大了又总感觉浪费，尽管不会实质分配这么大的物理内存。基于这么一个可能的原因，于是metaspace出现了，希望内存的管理不再受到限制，也不要怎么关注元数据这块的OOM问题，但是不意味着对应的OOM问题已经完全解决掉了

## Metaspace组成说明
### 组成
* Klass Metaspace
* NoKlass Metaspace
### 组成说明
* Klass Metaspace就是用来存klass的，klass就是class文件在jvm里的运行时数据结构。这块内存是紧接着Heap的，这块内存大小可通过-XX:CompressedClassSpaceSize参数来控制，这个参数默认是1G，但是这块内存也可以没有，假如没有开启压缩指针就不会有这块内存，这种情况下klass都会存在NoKlass Metaspace里，另外如果把-Xmx设置大于32G的话，其实也是没有这块内存的，因为会这么大内存会关闭压缩指针开关。还有就是这块内存最多只会存在一块。
* NoKlass Metaspace专门来存klass相关的其他的内容，比如method，constantPool等，这块内存是由多块内存组合起来的，所以可以认为是不连续的内存块组成的。这块内存是必须的，虽然叫做NoKlass Metaspace，但是也其实可以存klass的内容，上面已经提到了对应场景。
* Klass Metaspace和NoKlass Mestaspace都是所有classloader共享的，所以类加载器要分配内存，但是每个类加载器都有一个SpaceManager，来管理属于这个类加载的内存小块。如果Klass Metaspace用完了，那就会OOM了，不过一般情况下不会，NoKlass Mestaspace是由一块块内存慢慢组合起来的，在没有达到限制条件的情况下，会不断加长这条链，让它可以持续工作

## metaspace总结

### 引入Metaspace对PermGen配置影响
* 这里要注意MaxPermSize的区别，MaxMetaspaceSize并不会在jvm启动的时候分配一块这么大的内存出来，而MaxPermSize是会分配一块这么大的内存的
* VM的参数：PermSize 和 MaxPermSize 会被忽略并给出警告（如果在启用时设置了这两个参数）。

### Metaspace 内存分配模型
* 大部分类元数据并不在虚拟机，而是在本地内存分配。（这里的本地内存应该就是NativeMemory，供JVM进程使用）
* 用于描述类元数据的“klasses“已经被移除
* 分元数据分配了多个虚拟内存空间
* 给每个类加载器分配一个内存块的列表。块的大小取决于类加载器的类型; sun/反射/代理对应的类加载器的块会小一些
* 归还内存块，释放内存块列表
* 一旦元空间的数据被清空了，虚拟内存的空间会被回收掉
* 减少碎片的策略 
* Metaspace可以动态增长

### Metaspace 容量
&emsp;&emsp;新参数（MaxMetaspaceSize）用于限制本地内存分配给类元数据的大小。如果没有指定这个参数，元空间会在运行时根据需要动态调整。

### Metaspace垃圾回收
* 对于僵死的类及类加载器的垃圾回收将在元数据使用达到“MaxMetaspaceSize”参数的设定值时进行。
* 适时地监控和调整元空间对于减小垃圾回收频率和减少延时是很有必要的。持续的元空间垃圾回收说明，可能存在类、类加载器导致的内存泄漏或是大小设置不合适。

### Metaspace其他特点
* 充分利用了Java语言规范中的好处：类及相关的元数据的生命周期与类加载器的一致。
* 每个加载器有专门的存储空间
* 只进行线性分配
* 不会单独回收某个类
* 省掉了GC扫描及压缩的时间
* 元空间里的对象的位置是固定的
* 如果GC发现某个类加载器不再存活了，会把相关的空间整个回收掉

### Metaspace主要配置参数
* UseLargePagesInMetaspace
* InitialBootClassLoaderMetaspaceSize
* MetaspaceSize：初始化的Metaspace大小，控制元空间发生GC的阈值。GC后，动态增加或降低MetaspaceSize。在默认情况下，这个值大小根据不同的平台在12M到20M浮动。使用Java -XX:+PrintFlagsInitial命令查看本机的初始化参数。
* MaxMetaspaceSize：限制Metaspace增长的上限，防止因为某些情况导致Metaspace无限的使用本地内存，影响到其他程序。在本机上该参数的默认值为4294967295B（大约4096MB）。
* CompressedClassSpaceSize：默认1G，这个参数主要是设置Klass Metaspace的大小，不过这个参数设置了也不一定起作用，前提是能开启压缩指针，假如-Xmx超过了32G，压缩指针是开启不来的。如果有Klass Metaspace，那这块内存是和Heap连着的
* MinMetaspaceExpansion：
* MaxMetaspaceExpansion
* MinMetaspaceFreeRatio
* MaxMetaspaceFreeRatio







# 参考文档

* [JVM源码分析之Metaspace解密](http://lovestblog.cn/blog/2016/10/29/metaspace/)

* [什么是Java的永久代（PermGen）内存泄漏](https://www.aliyun.com/jiaocheng/284064.html)

* [JDK8-废弃永久代（PermGen）迎来元空间（Metaspace）](https://www.cnblogs.com/dennyzhangdd/p/6770188.html)

* [java case 3:方法区(PermGen)内存快速飙升问](https://blog.csdn.net/cza55007/article/details/46040351)

* [Java8内存模型—永久代(PermGen)和元空间(Metaspace)](https://www.cnblogs.com/paddix/p/5309550.html)









