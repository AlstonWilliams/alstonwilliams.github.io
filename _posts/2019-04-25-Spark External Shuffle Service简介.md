---
layout: post
title: Spark External Shuffle Service Timeout
date: 2019-04-25
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近遇到了一个问题,如下所示:
~~~
WARN TaskSetManager: Lost task 38.0 in stage 27.0 (TID 127459, linux-162): FetchFailed(BlockManagerId(66, 172.168.100.12, 23028), shuffleId=9, mapId=55, reduceId=38, message=
org.apache.spark.shuffle.FetchFailedException: java.util.concurrent.TimeoutException: Timeout waiting for task.
        at org.apache.spark.storage.ShuffleBlockFetcherIterator.throwFetchFailedException(ShuffleBlockFetcherIterator.scala:321)
        at org.apache.spark.storage.ShuffleBlockFetcherIterator.next(ShuffleBlockFetcherIterator.scala:306)
        at org.apache.spark.storage.ShuffleBlockFetcherIterator.next(ShuffleBlockFetcherIterator.scala:51)
        at scala.collection.Iterator$$anon$11.next(Iterator.scala:328)
        at scala.collection.Iterator$$anon$13.hasNext(Iterator.scala:371)
        at scala.collection.Iterator$$anon$11.hasNext(Iterator.scala:327)
        at org.apache.spark.util.CompletionIterator.hasNext(CompletionIterator.scala:32)
        at org.apache.spark.InterruptibleIterator.hasNext(InterruptibleIterator.scala:39)
        at scala.collection.Iterator$$anon$11.hasNext(Iterator.scala:327)
        at org.apache.spark.sql.execution.UnsafeExternalRowSorter.sort(UnsafeExternalRowSorter.java:173)
        at org.apache.spark.sql.execution.TungstenSort.org$apache$spark$sql$execution$TungstenSort$$executePartition$1(sort.scala:160)
        at org.apache.spark.sql.execution.TungstenSort$$anonfun$doExecute$4.apply(sort.scala:169)
        at org.apache.spark.sql.execution.TungstenSort$$anonfun$doExecute$4.apply(sort.scala:169)
        at org.apache.spark.rdd.MapPartitionsWithPreparationRDD.compute(MapPartitionsWithPreparationRDD.scala:64)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.ZippedPartitionsRDD2.compute(ZippedPartitionsRDD.scala:99)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsWithPreparationRDD.compute(MapPartitionsWithPreparationRDD.scala:63)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
        at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
        at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
        at org.apache.spark.scheduler.ShuffleMapTask.runTask(ShuffleMapTask.scala:73)
        at org.apache.spark.scheduler.ShuffleMapTask.runTask(ShuffleMapTask.scala:41)
        at org.apache.spark.scheduler.Task.run(Task.scala:88)
        at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.RuntimeException: java.util.concurrent.TimeoutException: Timeout waiting for task.
        at org.spark-project.guava.base.Throwables.propagate(Throwables.java:160)
        at org.apache.spark.network.client.TransportClient.sendRpcSync(TransportClient.java:196)
        at org.apache.spark.network.sasl.SaslClientBootstrap.doBootstrap(SaslClientBootstrap.java:76)
        at org.apache.spark.network.client.TransportClientFactory.createClient(TransportClientFactory.java:205)
        at org.apache.spark.network.client.TransportClientFactory.createClient(TransportClientFactory.java:156)
        at org.apache.spark.network.netty.NettyBlockTransferService$$anon$1.createAndStart(NettyBlockTransferService.scala:88)
        at org.apache.spark.network.shuffle.RetryingBlockFetcher.fetchAllOutstanding(RetryingBlockFetcher.java:140)
        at org.apache.spark.network.shuffle.RetryingBlockFetcher.start(RetryingBlockFetcher.java:120)
        at org.apache.spark.network.netty.NettyBlockTransferService.fetchBlocks(NettyBlockTransferService.scala:97)
        at org.apache.spark.storage.ShuffleBlockFetcherIterator.sendRequest(ShuffleBlockFetcherIterator.scala:152)
        at org.apache.spark.storage.ShuffleBlockFetcherIterator.next(ShuffleBlockFetcherIterator.scala:301)
        ... 57 more
Caused by: java.util.concurrent.TimeoutException: Timeout waiting for task.
        at org.spark-project.guava.util.concurrent.AbstractFuture$Sync.get(AbstractFuture.java:276)
        at org.spark-project.guava.util.concurrent.AbstractFuture.get(AbstractFuture.java:96)
        at org.apache.spark.network.client.TransportClient.sendRpcSync(TransportClient.java:192)
        ... 66 more
~~~

按理说一个Warning也没什么问题，但是这个Warning在好多Executor上出现，并最终导致我们的任务失败，这就有理由来深究一下了。

我们先来介绍一下什么是External Shuffle Service。这东西就是在NodeManager启动的一个Service，如果看过YARN的源码，会知道它里面有AUXService这东西，可以通过这东西对YARN进行扩展。而Spark呢，为了实现资源的动态分配，就实现了这么一个叫做External Shuffle Service的AUXService。它的作用就是，Executor算完以后，将输出文件的位置发送到External Shuffle Service，然后Executor就可以被回收了，然后其它的Executor启动时，去对应的External Shuffle Service找需要fetch的文件的位置。

知道它的原理，我们调试起来就很简单了，最可能的原因就是，External Shuffle Service压根没开启。

但是在我们的集群中，它确实是开启了的。

那是什么原因呢？

如果是External Shuffle Service GC的时间太长，导致没有及时响应Executor呢？有这种可能。

然后我们减少了数据量并重新跑了一下这个任务，就OK了。

## 参考资料
- [Spark doc](https://spark.apache.org/docs/latest/job-scheduling.html)
- [timeout-in-shuffle-problem-td16088](http://apache-spark-developers-list.1001551.n3.nabble.com/timeout-in-shuffle-problem-td16088.html)
- [external-shuffle-service-apache-spark](https://www.waitingforcode.com/apache-spark/external-shuffle-service-apache-spark/read)
