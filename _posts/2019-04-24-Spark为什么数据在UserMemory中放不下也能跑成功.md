---
layout: post
title: Spark为什么数据在UserMemory中放不下也能跑成功?
date: 2019-04-24
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近在做一件很蛋疼的工作，就是将HBase中的Bitmap，其中存的Hash都拿出来，然后做一下处理，得到我们想要的结果。

处理起来非常蛋疼，我们的Bitmap的最大尺寸是128MB，经过处理以后，数据量很可能就变成了10GB+,而我们又需要读全量的HBase表来处理。

本来我的方案是按照时间段来处理，我先取了某个时间段的，然后跑起来发现没什么问题，数据量也不小。Shuffle Spill(Memory)足足有几十T。看到这个数据量就有些疑问，为什么我只分配了1T的内存，这么大的数据量竟然能跑起来呢？我知道中间会有Spill To Disk的过程，可是到底哪时候发生呢?

带着这个问题，我开始探究Spark的执行计划。

首先，我们有如下代码:
~~~

import org.apache.spark.{SparkConf, SparkContext}

import scala.collection.mutable

object TestSparkExecutionPlan {

    def main(args: Array[String]): Unit = {

        val sparkConf = new SparkConf()
            .setMaster("local")
            .setAppName("Test")
        val sparkContext = new SparkContext(sparkConf)

        val rdd = sparkContext.parallelize(Seq("1", "2", "3"))
        rdd.repartition(1).map {
            case item => {
                item -> item // Breakpoint here
            }
        }.aggregateByKey(mutable.Set[String]())(
            (result: mutable.Set[String], item: String) => {
                result += item // Breakpoint here
            },
            (result1: mutable.Set[String], result2: mutable.Set[String]) => {
                result1 ++= result2
            }
        ).foreach(println(_))

    }

}
~~~

在我以前的认知中，我以为上面的`map`和`aggregateByKey`的执行过程，是对所有数据先执行`map`，然后对所有数据执行`aggregateByKey`，但是通过调试我发现这是错的。

在上述代码中的两处打断点，会发现运行的过程，是先对每条记录执行`map`，然后再对每条记录执行`aggregateByKey`。这样子就能解释上面我们提到那个问题了，即为什么少内存依然能处理大的数据量。

这都隐藏在`aggregateByKey`的实现中，我们来看它的内部实现:
~~~
/**
  * Insert the given iterator of keys and values into the map.
  *
  * When the underlying map needs to grow, check if the global pool of shuffle memory has
  * enough room for this to happen. If so, allocate the memory required to grow the map;
  * otherwise, spill the in-memory map to disk.
  *
  * The shuffle memory usage of the first trackMemoryThreshold entries is not tracked.
  */
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

在我以前的文章中，我提到过，ExecutionMemory，当Spark执行`*ByKey()`操作的时候，其中用到的`AppendOnlyMap`这个结构会使用这块内存。上述代码即是`ExternalAppendOnlyMap.insertAll`方法的实现。

在其`ExternalAppendOnlyMap.maybeSpill`方法中，就会判断是否需要进行Spill，如果需要的话，就Spill并清空当前`currentMap`:
~~~
/**
 * Spills the current in-memory collection to disk if needed. Attempts to acquire more
 * memory before spilling.
 *
 * @param collection collection to spill to disk
 * @param currentMemory estimated size of the collection in bytes
 * @return true if `collection` was spilled to disk; false otherwise
 */
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

而`map`完的数据，会逐条进入到这个方法中，当然，有一些情况可能需要Shuffle,这里先忽略这种情况。

既然逐条进入到这个方法中，那保存到`currentMap`以后，理论上来说，`map`方法占用的JVM内存其实就可以释放了，而`ExternalAppenedOnlyMap`由于在占用内存过多后，会自动进行Spill，所以保证了数据量大的情况下，我们分配的内存小也依然能够正常执行，只是会慢一点。

所以有的时候其实Executor由于内存原因失败并不一定就是我们分配的内存不够用。

在这个程序选取另一个时间段执行时，发现了一个现象，就是出现了OOM，提示为`Can't allocate memory....`。而看此Executor的Shuffle Spill(Memory)记录，发现实际上用了很少的内存，跟其它Executor完全不能比。后来登陆上这台Executor一看，发现是机器上的内存不够用了。

这里也可以看到，如果是出现了OOM异常，一般是我们分配的内存不够，或者没有正确衡量Spark对各个区域的使用而导致不均衡，如果是出现了`Can't allocate memory....`,则应当是本机上的内存不够用了。
