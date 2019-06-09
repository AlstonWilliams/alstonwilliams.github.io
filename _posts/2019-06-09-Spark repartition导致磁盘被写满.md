---
layout: post
title: Spark repartition导致磁盘被写满
date: 2019-06-09
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

在客户环境跑数据的时候，老是发现数据量稍大的时候，会把磁盘空间跑满，进一步导致任务失败。

而观察下来，发现是在`repartition`的时候，出现的问题。`repartition`的shuffle read是100G，shuffle write是400G。

回想起以前写的[Spark架构-Shuffle(译)
](https://alstonwilliams.github.io/spark/2019/02/17/Spark%E6%9E%B6%E6%9E%84-Shuffle(%E8%AF%91)/)这篇文章，shuffle的时候会写磁盘，所以猜到是这儿的原因。

于是把repartition操作去掉，直接写Hive，就没有问题了.
