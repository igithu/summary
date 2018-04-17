---
layout: post
title: "HBase RegionServer宕机恢复总结"
date: 2018-04-14
tags: hbase
---

# 背景
&emsp;&emsp;HBase宕机恢复是HBase一个重要部分，依靠这样一个宕机恢复机制，能够完成保证数据不丢失

# HLog的生命周期
&emsp;&emsp;宕机恢复实际上是依靠HLog进行的 所以HLog的声明周期 生成，使用，以至于最后被删除掉是很重要的 这里首先梳理HLog在正常情况的生命周期
## HLog生成
&emsp;&emsp;在写入的时候 RegionServer就会生成HLog，同时将其进行写入文件系统中
## HLog Roll
## HLog失效
## HLog清除


# 参考文档
* [HBase源码：HMaster启动过程](https://yq.aliyun.com/articles/25837)

