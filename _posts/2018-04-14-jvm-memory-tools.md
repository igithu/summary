---
layout: post
title: "Java内存问题排查工具总结"
date: 2018-04-14
tags: jvm
---

# JDK自带命令
* jmap：
** jmap -heap $pid, 查看java 堆内是否异常
** jmap -histo  $pid。查看对象内训使用情况
** jmap -dump:format=b,file=heap.bin $pid && jhat -J-mx768m -port <端口号:默认为7000> heap.bin.  对java做了一次heap dump，jhat之后再web中查看内存使用情况
* jstack：严格意义上不算是排查内存工具，jstack可以查看当前进程线程运行情况，内存方面 可以复查一下stack dump里的global jni reference是否一直在增长，如果一直在增长，jni调用可能存在内存溢出


# NativeMemoryTracking
&emsp;&emsp;NMT(NativeMemoryTracking)是HotSpot JVM提供跟踪内存的工具，配置到当前环境变量后，可以用jcmd就可以跟踪当前JVM进程内存的事情情况；根据[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/nmt-8.html)，目前的发布的版本不支持跟踪第三方Native code使用的内存。
下面只是做简单介绍，具体查看文档

## 配置
* 在环境变量中配置XX:NativeMemoryTracking=detail。
* 重启目标进程

## 使用
* 执行命令 jcmd $pid VM.native_memory detail > detail.log
* 查看detail.log信息


# PMAP


/usr/local/bin/pprof --text /opt/taobao/java/bin/java /tmp/alisenMonitor/tcmalloc.prof_32721.0291.heap > stack.log
