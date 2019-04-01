---
layout: post
title: HDFS Centralized Cache对MapReduce的影响
date: 2019-04-01
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
# HDFS Centralized Cache对MapReduce的影响

在[YARN源码解析(5)-MapReduce中，在数据本地节点分配Task是如何做到的？](https://alstonwilliams.github.io/hadoop/2019/02/17/YARN%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(5)-MapReduce%E4%B8%AD-%E5%9C%A8%E6%95%B0%E6%8D%AE%E6%9C%AC%E5%9C%B0%E8%8A%82%E7%82%B9%E5%88%86%E9%85%8DTask%E6%98%AF%E5%A6%82%E4%BD%95%E5%81%9A%E5%88%B0%E7%9A%84/)中，我们介绍了MapReduce如何在block所在的Host上分配Mapper的。

而了解了HDFS Centrailzed Cache以后，我们就有一个疑问了,Mapper的分配，会考虑HDFS Centrailized Cache么？

很遗憾，并没有。具体的Mapper分配流程请看上面提到的那篇文章。

那HDFS Centralized Cache怎么会提升性能呢？

这是因为，在Task运行的时候，会通过DFSClient来读取数据，而DFSClient中会检查，当前Task所在的主机是否存在这个block的Cache,如果有的话，直接从Cache读。

代码很简单，顺着Mapper的InputFormat往下看就找到了。时间关系，这里不在介绍。
