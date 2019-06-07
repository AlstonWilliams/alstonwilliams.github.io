---
layout: post
title: Spark coalesce的坑
date: 2019-06-07
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近在跑一个任务的时候，发现读取数据那一步总会卡死。

看代码，发现调用了`coalesce`函数。看此函数的注释:
~~~
/**
 * Return a new RDD that is reduced into `numPartitions` partitions.
 *
 * This results in a narrow dependency, e.g. if you go from 1000 partitions
 * to 100 partitions, there will not be a shuffle, instead each of the 100
 * new partitions will claim 10 of the current partitions.
 *
 * However, if you're doing a drastic coalesce, e.g. to numPartitions = 1,
 * this may result in your computation taking place on fewer nodes than
 * you like (e.g. one node in the case of numPartitions = 1). To avoid this,
 * you can pass shuffle = true. This will add a shuffle step, but means the
 * current upstream partitions will be executed in parallel (per whatever
 * the current partitioning is).
 *
 * @note With shuffle = true, you can actually coalesce to a larger number
 * of partitions. This is useful if you have a small number of partitions,
 * say 100, potentially with a few partitions being abnormally large. Calling
 * coalesce(1000, shuffle = true) will result in 1000 partitions with the
 * data distributed using a hash partitioner. The optional partition coalescer
 * passed in must be serializable.
 */
~~~

我们可以看到，只有当`shuffle`这个参数设置为true的时候，从少分区到多分区才真正起作用。

难怪我们设置了分区数量是1024，依旧会卡死。

于是我们换了个函数，调用`repartition`，就读得很快了。

所以需要注意，`coalesce`一般是用在缩减分区数量的场景，它会声明一个分区对应哪几个分区，避免不必要的shuffle。要扩大分区数量，一般不要调用此函数。
