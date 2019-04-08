---
layout: post
title: "kafka vs flume"
date: 2019-04-08
tags: kafka, flume
---


Kafka 与 Flume 很多功能确实是重复的。以下是评估两个系统的一些建议：
* Kafka 是一个通用型系统。你可以有许多的生产者和消费者分享多个主题。相反地，Flume 被设计成特定用途的工作，特定地向 HDFS 和 HBase 发送出去。Flume 为了更好地为 HDFS 服务而做了特定的优化，并且与 Hadoop 的安全体系整合在了一起。基于这样的结论，Hadoop 开发商 Cloudera 推荐如果数据需要被多个应用程序消费的话，推荐使用 Kafka，如果数据只是面向 Hadoop 的，可以使用 Flume。
* Flume 拥有许多配置的来源 (sources) 和存储池 (sinks)。然后，Kafka 拥有的是非常小的生产者和消费者环境体系，Kafka 社区并不是非常支持这样。如果你的数据来源已经确定，不需要额外的编码，那你可以使用 Flume 提供的 sources 和 sinks，反之，如果你需要准备自己的生产者和消费者，那你需要使用 Kafka。
* Flume 可以在拦截器里面实时处理数据。这个特性对于过滤数据非常有用。Kafka 需要一个外部系统帮助处理数据。
* 无论是 Kafka 或是 Flume，两个系统都可以保证不丢失数据。然后，Flume 不会复制事件。相应地，即使我们正在使用一个可以信赖的文件通道，如果 Flume agent 所在的这个节点宕机了，你会失去所有的事件访问能力直到你修复这个受损的节点。使用 Kafka 的管道特性不会有这样的问题。
* Flume 和 Kafka 可以一起工作的。如果你需要把流式数据从 Kafka 转移到 Hadoop，可以使用 Flume 代理 (agent)，将 kafka 当作一个来源 (source)，这样可以从 Kafka 读取数据到 Hadoop。你不需要去开发自己的消费者，你可以使用 Flume 与 Hadoop、HBase 相结合的特性，使用 Cloudera Manager 平台监控消费者，并且通过增加过滤器的方式处理数据。
