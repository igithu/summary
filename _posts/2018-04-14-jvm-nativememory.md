---
layout: post
title: "JVM与Java NativeMemory"
date: 2018-04-14
tags: jvm
---

# JVM

&emsp;&emsp;JVM使用的内存笼统上划分总共分为：Heap Memory和Native Memory(很多文档都说Heap Memory，但是我理解这是个广义泛指 应该包括Java heap，方法区， 计数器，本地方法栈等）
&emsp;&emsp;Heap Memory及其内部各组成的大小可以通过JVM的一系列命令行参数来控制 Native Memory也称为C-Heap，是供JVM自身进程使用的。
&emsp;&emsp;Native Memory没有相应的参数来控制大小，其大小依赖于操作系统进程的最大值（对于32位系统就是3~4G，各种系统的实现并不一样），以及生成的Java字节码大小、创建的线程数量、维持java对象的状态信息大小（用于GC）以及一些第三方的包，比如JDBC驱动使用的native内存

![image01](https://igithu.github.io/summary/images/jvm-nm.jpg)

# Native Memory

* 管理java heap的状态数据（用于GC）;
* JNI调用，也就是Native Stack，底层如果与C++对接，Java层会通过JNI与C++进行交互;
* JIT（即使编译器）编译时使用Native Memory，并且JIT的输入（Java字节码）和输出（可执行代码）也都是保存在Native Memory；
* NIO direct buffer。对于IBM JVM和Hotspot，都可以通过-XX:MaxDirectMemorySize来设置nio直接缓冲区的最大值。默认是64M。超过这个时，会按照32M自动增大。
* 对于IBM的JVM某些版本实现，类加载器和类信息都是保存在Native Memory中的。


# DirectBuffer

&emsp;&emsp;使用DirectBuffer可用通过调用函数ByteBuffer.allocateDirect() 来进行获取，使用DirectBuffer整体上访问更快，避免了从HeapBuffer还需要从java堆拷贝到本地堆，操作系统直接访问的是DirectBuffer。DirectBuffer对象的数据实际是保存在native heap中，换句话说：DirectBuffer是分成两个部分 一部分是在JVM Heap Memory内部 主要存储的地址address（DirectBuffer自身是在Java 堆内的），另一部数据部分是存储Native Memory主要是通过底层malloc进行分配的（这部分并不是内核态，而是处在用户态上）

&emsp;&emsp;但是引用保存在HeapBuffer中。 另外，DirectBuffer的引用是直接分配在堆得Old区的，因此其回收时机是在FullGC时。因此，需要避免频繁的分配DirectBuffer，这样很容易导致Native Memory溢出。
  
  
  
  
  
  
  

# 参考文档

[JVM的Heap Memory和Native Memory](http://mahaijin.github.io/2015/04/27/JVM%E7%9A%84Heap%20Memory%E5%92%8CNative%20Memory/)

[Java NIO中，关于DirectBuffer，HeapBuffer的疑问？](https://www.zhihu.com/question/57374068)
