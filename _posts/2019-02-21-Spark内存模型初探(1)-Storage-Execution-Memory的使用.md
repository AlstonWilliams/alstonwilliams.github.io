---
layout: post
title: Spark内存模型初探(1)-Storage/Execution Memory的使用
date: 2019-02-21
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
过去，我翻译了几篇关于Spark内存模型的文章。翻译完以后，我觉得我对Spark内存模型已经够理解了，可是，纸上得来终觉浅，实际跑Spark任务的时候，还是会遇到OOM，而我并不知道是哪部分发生了OOM，也就不知道该如何分配Storage Memory/Execution Memory等，才会保证资源不会被浪费，也不会太小导致资源不足。

正是出于这一点，我开始深入理解Spark的内存模型。我想知道Storage Memory，Execution Memory里面到底放的是什么，User Memory里面到底放的是什么。进一步明确我该如何对这两者进行分配。

本文中，主要会介绍Storage Memory以及Execution Memory中到底存放了什么，这是初步得出的结果，并不完善，文末我也会提出很多问题。但是这些问题，后面我会一点点来探索，一点点解答，并发布出来。

另外，由于作者在Spark领域也是初学者，所以请读者在阅读本文的时候，保持怀疑的态度，自己验证，探索。我会给出我的测试代码。

另外，本文介绍的都是针对Spark-core的，并没有涉及Spark-SQL。其实Spark-SQL我们用的也蛮多的，有时间也应该认真探索一下。

## 环境

Spark版本: Spark 2.4.0-SNAPSHOT
Java版本: Java 8
操作系统: Ubuntu 17.10

运行的Spark为standalone模式，本机启动三个Worker。

由于Spark内部其实在内存分配时，并没有打印出来日志，包括debug日志，所以，我们需要向Spark中手动添加一些debug代码，这些我已经放在了这个测试项目的patch目录，需要读者合并并且重新编译一下。

## 项目地址

[TestBigDataProject](https://github.com/AlstonWilliams/TestBigDataProject)

## StorageMemory中存放的内容

StorageMemory分为两部分，一部分是Unroll Memory，另一部分就是剩下的Storage Memory。

其中，Unroll Memory，会被用于broadcast变量的展开。

当我们调用`RDD.persist()`或者`RDD.cache()`时，这些数据会被存储到Storage Memory中。

详细的代码请参考这两个类:
- com.hypers.spark.TestSparkBroadcastMemoryUsage
- com.hypers.spark.TestSparkCacheMemoryUsage

Storage  Memory的分配很简单，重新编译Spark以后，会在Executor的日志中，看到相应的日志。

## ExecutionMemory中存放的内容

ExecutionMemory这个调试起来就比较困难了。因为我得到的信息，一直都是它用于存放shuffle时的中间结果，然后我用`repartition()`函数进行测试，一直没有看到有分配ExecutionMemory。emmmm......

后来想到，干脆去源码里看看，到底什么时候会分配ExecutionMemory。

然后，追本溯源，发现只有一个地方会调用它，相关代码(Spillable.maybeSpill(collection: C, currentMemory: Long))如下:
~~~
protected def maybeSpill(collection: C, currentMemory: Long): Boolean = {

    // >>>>>>>>>>>>>>>>>>>
    log.debug("---------- calling maybeSpill, currentMemory: " + currentMemory)
    log.debug("---------- _elementRead: " + _elementsRead)
    log.debug("---------- numElementsForceSpillThreadshold: " + numElementsForceSpillThreshold)
    log.debug("---------- elementReads: " + elementsRead)
    log.debug("---------- myMemoryThreshold: " + myMemoryThreshold)
    // <<<<<<<<<<<<<<<<<<<

    var shouldSpill = false
    if (elementsRead % 32 == 0 && currentMemory >= myMemoryThreshold) {
      // Claim up to double our current memory from the shuffle memory pool
      val amountToRequest = 2 * currentMemory - myMemoryThreshold
      val granted = acquireMemory(amountToRequest)
      myMemoryThreshold += granted
      // If we were granted too little memory to grow further (either tryToAcquire returned 0,
      // or we already had more memory than myMemoryThreshold), spill the current collection
      shouldSpill = currentMemory >= myMemoryThreshold
    }
    shouldSpill = shouldSpill || _elementsRead > numElementsForceSpillThreshold

    // >>>>>>>>>>>>>>>>>>>
    log.debug("--------- shouldSpill: " + shouldSpill)
    // <<<<<<<<<<<<<<<<<<<

    // Actually spill
    if (shouldSpill) {
      _spillCount += 1
      logSpillage(currentMemory)
      spill(collection)
      _elementsRead = 0
      _memoryBytesSpilled += currentMemory
      releaseMemory()
    }
    shouldSpill
}
~~~

其中的`acquireMemory()`方法就是专门用来申请Execution Memory的。而`maybeSpill(collection: C, currentMemory: Long)`这个方法，会被`ExternalAppendOnlyMap`以及`ExternalSorter`调用。我只看过`ExternalAppendOnlyMap`相关的代码，所以这里只拿它举例:
~~~
  def insertAll(entries: Iterator[Product2[K, V]]): Unit = {

    // >>>>>>>>>>>>>>>>>>>
    log.debug("-------------- calling insertAll")
    // <<<<<<<<<<<<<<<<<<<

    if (currentMap == null) {
      throw new IllegalStateException(
        "Cannot insert new elements into a map after calling iterator")
    }
    // An update function for the map that we reuse across entries to avoid allocating
    // a new closure each time
    var curEntry: Product2[K, V] = null
    val update: (Boolean, C) => C = (hadVal, oldVal) => {
      if (hadVal) mergeValue(oldVal, curEntry._2) else createCombiner(curEntry._2)
    }

    while (entries.hasNext) {
      curEntry = entries.next()
      val estimatedSize = currentMap.estimateSize()
      if (estimatedSize > _peakMemoryUsedBytes) {
        _peakMemoryUsedBytes = estimatedSize
      }
      if (maybeSpill(currentMap, estimatedSize)) {
        currentMap = new SizeTrackingAppendOnlyMap[K, C]
      }
      currentMap.changeValue(curEntry._1, update)
      addElementsRead()
    }
  }
~~~

我们可以看到，当插入到`ExternalAppendOnlyMap`时，会检测是否需要进行spill，如果自从上次spill以后，插入到`ExternalAppendOnlyMap`的entry的数量是32的倍数，并且当前`ExternalAppendOnlyMap`占用的内存已经超过了我们设置的阈值，那么就尝试分配ExecutionMemory。如果分配成功，那么OK,只要你没有设置强制spill的阈值，那么就不spill。否则的话，就做一次spill。

而什么地方会调用`ExternalAppendOnlyMap.insertAll(entries: Iterator[Product2[K, V]])`呢？

1. Spark 1.6以后的版本并且`spark.memory.useLegacyMode=false`。在Spark 1.6以后，默认是false。如果是true的话，会使用`AppendOnlyMap`
2. 当我们调用`RDD.*ByKey()`这些方法的时候。

关于第二条，在StackOverflow上有个用`reduceByKey`举的例子，请点击[这里](https://stackoverflow.com/questions/41585673/understanding-shuffle-managers-in-spark)查看。

测试代码请参考:
- com.hypers.spark.TestSparkAggregateByKeyMemoryUsage
- com.hypers.spark.TestSparkGroupByKeyMemoryUsage
- com.hypers.spark.TestSparkReduceByKeyMemoryUsage
- com.hypers.spark.TestSparkRepartitionMemoryUsage

需要注意的是，测试ExecutionMemory的时候，为了保证能够进入到申请ExecutionMemory的逻辑，我加了`spark.shuffle.spill.initialMemoryThreshold=1`这个设置。也就是说，只要`ExternalAppendOnlyMap`中数据的大小大于1Byte，并且上次spill以后到现在插入了32的倍数条数据，就需要可以申请ExecutionMemory。

## 疑问

在测试的过程中，不断冒出来新的疑问，等待我去解答:
- UserMemory是怎样被使用的?当我们在`RDD.map(function)`中的`function`中初始化新的对象时，是在哪部分内存被初始化的？是不是就是UserMemory?这部分想测试，但是无从下手。因为Spark的内存模型，在Java heap上不一定是连续的。
- 如果确实是在UserMemory上分配的，那么，我们知道理论上它的大小是`Java Heap - Reserved Memory - (Java Heap * spark.memory.fraction)`。可是，如果我`function`里就是要初始化很多个对象，超过了这个大小的话，Spark会怎样做？OOM么？但是此时并没有超过Java Heap的大小啊。而且，我并没有在Spark代码中看到跟UserMemory相关的代码。
- 测试的时候，偶尔会出现很诡异的现象。即使给Executor设置了`-Xmx1024M`，但是有时出现OOM的时候，查看GC日志，此时实际的Executor Heap才用了200M左右。会不会当时系统可用内存只有200M，所有导致Executor Heap不能自动扩容?
- 这是Executor在shuffle时遇到的问题，那么，shuffle时，Reducer端接收到的数据，是在哪个部分分配的?是不是就是UserMemory？
- 在Reducer端接收数据时，由于单条数据都很大，有50MB，会直接在老年代分配，而由于当时系统可用内存过小，老年代的内存不够，并且无法进行扩容了，所以导致了OOM。有没有可能是这种情况?
- Spark的内存模型，跟Java内存模型之间，是什么关系？
