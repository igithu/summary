---
layout: post
title: "Java内存问题排查工具总结"
date: 2018-04-14
tags: jvm
---

# JDK自带命令
## jmap
* `jmap -heap $pid`, 查看java 堆内是否异常
* `jmap -histo  $pid`, 查看对象内训使用情况
* `jmap -dump:format=b,file=heap.bin $pid && jhat -J-mx768m -port <端口号:默认为7000> heap.bin`.  对java做了一次heap dump，jhat之后再web中查看内存使用情况
## jstat
* 实时监视虚拟机运行时的类装载情况、各部分内存占用情况、GC情况、JIT编译情况等
* 命令：`jstat –gc $pid $interval` $查询次数
## jstack
* jstack：严格意义上不算是排查内存工具，jstack可以查看当前进程线程运行情况，内存方面 可以复查一下stack dump里的global jni reference是否一直在增长，如果一直在增长，jni调用可能存在内存溢出,
## GC日志
### GC日志的输出

### GC日志


JVM内存结构（来源网络）

![image01](https://igithu.github.io/summary/images/jvm-heap.jpg)
  
    
     
# NativeMemoryTracking
&emsp;&emsp;NMT(`NativeMemoryTracking`)是HotSpot JVM提供跟踪内存的工具，配置到当前环境变量后，可以用jcmd就可以跟踪当前JVM进程内存的事情情况；根据[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html)，目前的发布的版本不支持跟踪第三方Native code使用的内存。
下面只是做简单介绍，具体查看文档

## 配置
* 在环境变量中配置`-XX:NativeMemoryTracking=detail`
* 重启目标进程

## 使用
* 执行命令 `jcmd $pid VM.native_memory detail > detail.log`
* 查看detail.log信息   
   
      
      
# PMAP
&emsp;&emsp;pmap命令用于报告`进程的内存映射关系`，是Linux调试及运维一个很好的工具。pmap效果
![image02](https://igithu.github.io/summary/images/pmap.png)


## 选项
* -x：显示扩展格式；
* -d：显示设备格式；
* -q：不显示头尾行；
* -V：显示指定版本。

## 最后一行
* `mapped` 表示该进程映射的虚拟地址空间大小，也就是该进程预先分配的虚拟内存大小，即ps出的vsz
* `writeable/private` 表示进程所占用的私有地址空间大小，也就是该进程实际使用的内存大小     
* `shared` 表示进程和其他进程共享的内存大小

## 关于anon
&emsp;&emsp;这里面特意查了下，根据Stack Overflow上面的解释很多：`anon实际上是通过malloc或者mmap底层函数调用接口进行内存分配的`，如果看到很多分配内存大小一样的anon，一般都是在线程栈里面也会分配的内存部分   
   
      
      
# 参考文档

* [记一次java native memory增长问题的排查](http://blog.2baxb.me/archives/918)
* [进程物理内存远大于Xmx的问题分析](http://lovestblog.cn/blog/2015/08/21/rssxmx/)
* [gperftools做堆外内存分析（案例JVM Inflater 内存泄漏分析）](http://www.dylan326.com/2017/09/28/gperftools/)
* [Trying to locate a leak! What does anon mean for pmap?](https://stackoverflow.com/questions/1477885/trying-to-locate-a-leak-what-does-anon-mean-for-pmap)
