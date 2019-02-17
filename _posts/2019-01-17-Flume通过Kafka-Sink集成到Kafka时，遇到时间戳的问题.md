---
layout: post
title: Flume通过Kafka-Sink集成到Kafka时，遇到时间戳的问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
因为我们需要将日志文件发送给Kafka，然后通过其流处理的功能对日志进行加工，所以需要通过Kafka Sink集成到Kafka.

然而，当集成时，发现了一个问题．我用的Kafka版本是0.11.0.0,Flume的版本是1.7.0.

如果你跑着一个Kafka流处理应用，你会发现它会提示下面的错误:

**Possibly because a pre-0.10 producer client was used to write this record to Kafka without embedding a timestamp, or because the input topic was created before upgrading the Kafka cluster to 0.10+. Use a different TimestampExtractor to process this data.**

那这个错误是由什么造成的呢?

**This error means that the timestamp extractor of your Kafka Streams application failed to extract a valid timestamp from a record. Typically, this points to a problem with the record (e.g., the record does not contain a timestamp at all), but it could also indicate a problem or bug in the timestamp extractor used by the application.**

具体的关于这个错误的分析，请点击[这个链接](http://docs.confluent.io/current/streams/faq.html)

解决的方案也很简单，把Flume的依赖中的**kafka-clients-0.9.0.1.jar**替换为**kafka-clients-0.11.0.0.jar**就行了．如果你是手动编译的Flume，那么其依赖位于**apache-flume-1.7.0-src/flume-ng-dist/target/apache-flume-1.7.0-bin/apache-flume-1.7.0-bin/lib**．如果你是下载的可以直接用的版本，那么它的依赖位于哪我也不清楚，各位自行搜索一下吧．
