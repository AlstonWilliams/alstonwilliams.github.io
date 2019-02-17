---
layout: post
title: Spark中reduceByKey()和groupByKey()的区别（译）
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
我们都知道，在Spark当中，分组操作时，提供了这么两个函数。那么，这两个方法有什么区别呢？我们应该使用哪一个呢？

我们用WordCount程序来举例。

~~~~
val words = Array("one", "two", "two", "three", "three", "three")
val wordPairsRDD = sc.parallelize(words).map(word => (word, 1))

val wordCountsWithReduce = wordPairsRDD
  .reduceByKey(_ + _)
  .collect()

val wordCountsWithGroup = wordPairsRDD
  .groupByKey()
  .map(t => (t._1, t._2.sum))
  .collect()
~~~~

这两种做法的结果都是正确的。

但是，在大的数据集上，**reduceByKey()**的效果比**groupByKey()**的效果更好一些。因为**reduceByKey()**会在shuffle之前对数据进行合并。

下面一张图就能表示**reduceByKey()**都做了什么。

![](http://upload-images.jianshu.io/upload_images/4108852-b3878158fad026b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而当我们调用**reduceByKey()**的时候，所有的键值对都会被shuffle到下一个stage，传输的数据比较多，自然效率低一些。

![](http://upload-images.jianshu.io/upload_images/4108852-b9bee40f89a4db5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 原文链接

[Prefer reduceByKye() over groupByKey()](https://databricks.gitbooks.io/databricks-spark-knowledge-base/content/best_practices/prefer_reducebykey_over_groupbykey.html)
