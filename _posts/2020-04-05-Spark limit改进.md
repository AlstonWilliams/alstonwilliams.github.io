---
layout: post
title: Spark limit改进
date: 2020-04-12
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

在执行一条SQL，类似sparkSession.sql(“select * from table where id=1 limit 10000000”)这种SQL时，发现速度很慢。后面只有一个partition在处理。

limit的原理就是在先根据查询条件组成一个RDD，然后每个partition取limit数量，再统一发给一个partition，然后取出limit数量的Row.

这个实现原理，对于那些有order by的情景来说，非常合理。但是对于我们这种只需要取很多数量的行数的情景，没有order by的情景来说，就非常不合理。因为后面只有一个partition在处理，那万一我们取的行数多了，就很有可能造成内存溢出，导致任务失败。

所以，我们对任务做了一个优化。实现一个类似limit的算子。原理是先执行没有limit的语句，然后看有多少个partition，然后根据limit/partition决定每个partition保留特定行数的数据。代码如下：

~~~
val resultTemp: RDD[Row] = ss.sql(sql).rdd

val partitionNum = resultTemp.getNumPartitions

val eachRowPartitionNum = count.toInt / partitionNum

val limitedRDD = resultTemp.mapPartitions {
    case iterator => {
        iterator.take(eachRowPartitionNum)
    }
}
~~~

这样就实现limit类似的效果了。

但是这样做有一个缺陷：
1. order by的场景用不了
2. 并不是精确的limit，会少部分数据
