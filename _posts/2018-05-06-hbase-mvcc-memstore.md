---
layout: post
title: "从MVCC到MemStore "
date: 2018-05-16
tags: hbase
---

# 背景
&emsp;&emsp;在数据读写过程中，离开不了两样重要组件：mvcc和MemStore；其中mvcc是HBase事务保证的基础之一，MemStore则提供了重要读写内存介质。这里主要介绍两者如果完成各自任务
