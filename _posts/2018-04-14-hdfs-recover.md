---
layout: post
title: "HDFS 文件Recover总结"
date: 2018-04-14
tags: hdfs
---

# 背景
&emsp;&emsp;HDFS的Recover机制，主要针对异常文件关闭进行recover操作，目的是保证已经写入的数据的可读性，最大程度保证数据对外服务的稳定性；HDFS提供Lease Recovery, Block Recovery和Pipeline Recovery等Recovery机制用于容错
