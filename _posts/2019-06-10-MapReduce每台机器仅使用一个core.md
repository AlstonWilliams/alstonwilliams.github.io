---
layout: post
title: MapReduce每台机器仅使用一个core
date: 2019-06-10
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---

在客户环境跑数据时，发现跑得很慢，我们设置了共100个Reducer，客户只有八台机器，每次只有八个Reducer在运行，而且运行一批需要35分钟左右，速度特别慢。

查过资料得知，这是因为MapReduce会根据`机器上的内存数/每个Reducer需要的内存`，来决定每台机器上运行几个Reducer。而我们的机器，能够运行的内存只有20+G，我们给每个Reducer分配了10G，还有其它Spark任务占用了内存，就导致了只有8个Reducer在运行。

后来，将每个Reducer需要的内存调整成4G，并且设置`mapred.tasktracker.reduce.tasks.maximum`为5。就没有了这个问题。

参考资料:[Run Map-Reduce application on multiple core on the same machine](https://stackoverflow.com/questions/18810310/run-map-reduce-application-on-multiple-core-on-the-same-machine)
