---
layout: post
title: Spark架构-Shuffle(译)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
原文链接 https://0x0fff.com/spark-architecture-shuffle/

这是关于Spark 架构的第二篇文章。在这篇文章中，我会详细介绍关于Shuffle的事情。前一篇文章主要是关于Spark的总体架构，以及内存管理，你可以点击[spark-architecture](https://0x0fff.com/spark-architecture/)来查看。另一篇关于Spark内存管理的文章，你可以点击[spark-memory-management](https://0x0fff.com/spark-memory-management/)来查看.

![](https://upload-images.jianshu.io/upload_images/4108852-d13b6cc85ba30249.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(译者注：原文此处有关于Shuffle的简单介绍，主要是介绍它的含义以及意义。这个大家应该都知道，就不翻译了)

在这篇文章中，我会延续MapReduce中的命名约定。在Shuffle操作中，负责分发数据的Executor叫做`Mapper`，而接收数据的Executor叫做`Reducer`。

Shuffle时，有两个很重要的用于压缩的参数:
-  ***spark.shuffle.compress*** – 是否要Spark对Shuffle的输出进行压缩
- ***spark.shuffle.spill.compress*** – 是否压缩Shuffle中间的刷写到磁盘的文件

这两个参数默认都是true。并且，默认都会使用***spark.io.compression.codec***来压缩数据，默认情况下，是[snappy](https://en.wikipedia.org/wiki/Snappy_(software))

你可能知道，Spark中有好几个Shuffle的实现。那具体使用哪个Shuffle的实现，取决于`spark.shuffle.manager`这个参数。可能的选项是:`hash, sort, tungsten-sort`。从Spark 1.2.0开始，`sort`是默认选项。

## Hash Shuffle

Spark 1.2.0以前，这是默认使用的shuffle实现(***spark.shuffle.manager ****= hash*)。但是呢，第一版往往都是有弊端的。这不，这家伙因为每个Mapper都会给每个Reducer创建一个文件，就很容易造成[集群中创建了大量文件](http://www.cs.berkeley.edu/~kubitron/courses/cs262a-F13/projects/reports/project16_report.pdf)的事件。假设有`M`个Mapper，有`N`个Reducer，那集群中就会为Shuffle创建`M * R`个文件。如果Mappers和Reducers的数量够多，那么这是一件很恐怖的事情。这同时会导致更多的output buffer size，更多的文件句柄，以及创建和删除这些文件的开销。有一个演讲[how Yahoo faced all these problems](http://spark-summit.org/2013/wp-content/uploads/2013/10/Li-AEX-Spark-yahoo.pdf)，46K个mappers，46k个reducers，总共在集群中生成了20亿个文件。

Hash Shuffle的逻辑非常简单，就是Mapper计算Partition的数量，作为Reducer的数量，并且给每个Reducer创建一个文件，然后在output中循环，扫到一个就输出到对应的文件。

如下图所示:
![](https://upload-images.jianshu.io/upload_images/4108852-df4110239c9e2e13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决上面的那个问题，有一个优化的选项`spark.shuffle.consolidateFiles`(默认是false)。如果设置成true，Mapper的输出会被合并。假设集群中有**E**个executors(**“–num-executors”** for YARN)，每个executor都有**C**个cores(**“spark.executor.cores”** or **“–executor-cores”** for YARN) ，每个Task需要**T**个CPU(**“spark.task.cpus“**)，那集群中能执行的Task为**E * C / T**。总共要创建的文件数量是**E * (C / T) * R**（译者注,也就是**Task number * R**，为什么不是Mapper的数量呢?）。所以，如果有100个executor，每个executor10个cores,每个Task一个core,共有46000个reducers，那么，创建的文件的数量，将会从20亿降到4600w。这对性能是一个不小的提升。

这个选项的实现很直接，[FileShuffleBlockResolver](https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/shuffle/FileShuffleBlockResolver.scala)：它不会给每个Reducers创建一个文件，而是会创建一个文件池。当一个Map Task要开始输出数据时，它会从这个池中请求R个文件。输出完以后，再还给文件池。每个Executor只能同时执行**C / T**个Task，所以它只能创建**C / T**组输出文件，每一组有**R**个文件。

流程如下所示:

![](https://upload-images.jianshu.io/upload_images/4108852-540a33f4eb50266c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Hash Shuffle的优点:
- 非常快速。不需要排序，也不需要维护哈希表。没有给数据排序时的内存的开销。
- 没有额外的IO开销。数据只被读写一次。

Hash Shuffle的缺点:
- 当Reducer的数量很多时，很容易有性能问题
- 当大量文件被写到文件系统时，会产生大量的Random IO。而Random IO是最慢的，比Sequential IO要慢100倍。

关于在文件很多时，IO慢的问题，可以查看这篇文章 [millions of files on a single filesystem](http://events.linuxfoundation.org/slides/2010/linuxcon2010_wheeler.pdf)

当数据被写到文件时，会被序列化，并会根据你的配置决定是否进行压缩。而读的时候，则于此相反。在Reducer端，一个非常重要的配置是`spark.reducer.maxSizeInFlight`(默认是48MB)，它决定了每个Reducer从能够从远程Executors接收的数据量。如果你增大这个值，那么每次Reducers都会用更大的窗口来接收Mapper端发过来的数据，这会提高性能，但是同时，也会提高Reducer的内存使用量。

可以查看这个文档[Spark performance optimization: shuffle tuning 
](http://bigdatatn.blogspot.com/2017/05/spark-performance-optimization-shuffle.html)

如果Reducer不在乎Record的顺序，那么Reducer只会拿到它依赖的Mapper的的一个iterator。但是，如果在乎顺序，那么会拿到全部数据，并通过[ExternalSorter](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/util/collection/ExternalSorter.scala)在Reducer端做一次排序。

## Sort Shuffle

从Spark 1.2.0开始，这是Spark中默认的Shuffle算法(***spark.shuffle.manager= sort***)。这个算法，是对[Hadoop MapReduce](https://0x0fff.com/hadoop-mapreduce-comprehensive-description/)中的Sort Shuffle的一次模仿。在Hash Shuffle中，Mapper会给每个Reducer创建一个文件，但是在Sort Shuffle中，只创建一个文件，在其中根据Reducer id以及索引进行排序。如果要访问某个Reducer的数据，我们只需要知道它在这个文件中的位置，然后在`fread`之前做一次`fseek`即可。

当然，如果Reducer的数量少，那很明显，Hash到不同的文件，比仅有一个文件，然后通过sort将相同Reducer的数据放到一起，要快，所以，当Reducer的数量小于`spark.shuffle.sort.bypassMergeThreshold`（默认是200），那么还是会为每个Reducer生成一个文件，然后将这些文件合成一个文件。感兴趣的话，可以查看[BypassMergeSortShuffleWriter](https://github.com/apache/spark/blob/master/core/src/main/java/org/apache/spark/shuffle/sort/BypassMergeSortShuffleWriter.java)的更多细节。

这个实现中，很有趣的一点是，它仅仅在Mapper端对数据进行排序，而并不会在Reducer端，在接收时，按照这种顺序做一次合并。所以，当Reducer需要数据有序时，它需要重新做一次排序。Cloudera开始要做这件事情了:[http://blog.cloudera.com/blog/2015/01/improving-sort-performance-in-apache-spark-its-a-double/](http://blog.cloudera.com/blog/2015/01/improving-sort-performance-in-apache-spark-its-a-double/).

你可能知道，Reducer在排序的时候，使用的是[TimSort](https://en.wikipedia.org/wiki/Timsort), 这个算法会利用数据局部有序的特色。

(译者注：原文这里有对最小堆和TimSort的对比，译者对这些并不是很熟悉，所以就暂时不译了。感兴趣的读者可以自行阅读原文)

那如果Reducer并没有足够的内存，放下全部Mapper发过来的数据，咋办？这时候就需要将中间数据刷到磁盘上了。`spark.shuffle.spill`这个参数决定是否将中间结果刷到磁盘上，默认是开启的。如果你关闭了这个选项，然后Reducer没有足够的内存，放下Mapper发过来的数据，那就会碰到OOM。

Executor的堆内存中，可以用来存储Mapper端发过来的结果的内存有`JVM Heap Size * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction`这么多，默认是`JVM Heap Size * 0.2 * 0.8 = JVM Heap Size * 0.16`。需要注意的是，如果你在一个Executor中运行多个线程( setting the ratio of spark.executor.cores / spark.task.cpus to more than 1，即一个Executor有多个Task)，那么每个Task可以用的存储Mapper端结果的内存是`JVM Heap Size * spark.shuffle.memoryFraction * spark.shuffle.safetyFraction / (spark.executor.cores / spark.task.cpus)`。如果每个Executor有两个cores，其它选项默认的话，那么就是`0.08 * JVM Heap Size`

Spark内部使用[AppendOnlyMap](https://github.com/apache/spark/blob/branch-1.5/core/src/main/scala/org/apache/spark/util/collection/AppendOnlyMap.scala)来存储Mapper端发过来的结果。但是，Spark使用了他们自己实现的Hash table，在这个实现中，使用了开放定址法，并且会通过[quadratic probing](http://en.wikipedia.org/wiki/Quadratic_probing)把key和value存储到同一个array中。他们选择用Google Guava library中的[MurmurHash3](https://en.wikipedia.org/wiki/MurmurHash)这个Hash函数。

这个哈希表，允许Spark同时做聚集操作。对于每个key，value到来时，会跟以前的结果做一次聚集操作，然后存储为一个新的value。

当刷到磁盘时，会对AppendOnlyMap执行sort操作，它内部会调用TimSort，然后数据会被刷到磁盘上。

![](https://upload-images.jianshu.io/upload_images/4108852-b2725d2ead5782cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优点:
- Mapper创建的文件少
- Random IO更少，大多数都是sequential writes和reads

缺点:
- 排序比Hash慢。所以需要细心找出bypassMergeThreshold。因为默认值可能有点大
- 如果你使用SSD，那么Hash shuffle可能更好(译者注：为啥?因为Random IO代价更小么？但是创建了很多文件的事情没办法解决啊)

## Unsafe Shuffle or Tungsten Sort

我们可以通过Spark 1.4.0 以后的***spark.shuffle.manager = tungsten-sort***参数来启用这种Shuffle。它是project “Tungsten”](https://issues.apache.org/jira/browse/SPARK-7075)的一部分。创造这种Shuffle的初衷，可以[点击这里](https://issues.apache.org/jira/browse/SPARK-7081)查看。

这种Shuffle中，所做的优化如下:
1. 可以不经过反序列化而直接操纵数据。因为它内部使用的是unsafe (sun.misc.Unsafe) memory copy functions。

2. 内部对CPU cache做了优化。它在排序时，每个record只使用8 bytes，因为把record pointer以及partition id做了压缩。对应的排序算法是 [ShuffleExternalSorter](https://github.com/apache/spark/blob/master/core/src/main/java/org/apache/spark/shuffle/sort/ShuffleExternalSorter.java)

3. 将数据刷到磁盘上时，不需要经过反序列化(no deserialize-compare-serialize-spill logic)。

4. 如果压缩算法支持直接对序列化的stream进行拼接，那么，就可以使用spill-merge优化。现在只有Spark的LZF serializer支持直接对序列化之后的Stream进行拼接。并且只有在开起了***shuffle.unsafe.fastMergeEnabled***才起作用。

只有当下面的几个条件，全部满足的时候，才会使用这种Shuffle：
- 不是为了做聚集操作才做Shuffle。因为如果做聚集操作的话，很明显需要反序列化数据，并进行聚集。这样子的话，就失去了Unsafe Shuffle最重要的优势-直接对序列化后的数据进行操作。
- Shuffle serializer支持serialized values的重定位(译者注：没看懂)。当前只有KrySerializer以及Sparkr SQL的自定义serailizer支持。
- Shuffle产生的partition数量，小于16777216
- 序列化之后，每条record的大小，不能大于128MB

另外，有一点需要注意的是，在sort时，只会根据partition id 进行排序。也就是说，在Reducer端执行的merge pre-sorted data，以及依赖的TimSort算法，现在都将不起作用。

sort时，依靠8-byte values，每个value都会包含一个指向序列化以后数据的指针，以及partition id，这也是我们为什么说partition的数量不能超过16777216.(译者注：16777216是2**24，也就是说，这8-bytes里面，指针占5-byte，另外3-byte 才是partition id)

流程图如下:
![](https://upload-images.jianshu.io/upload_images/4108852-521e507ae38967b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

优点：
- 做了很多性能优化

缺点:
- 不会在Mapper端处理数据的顺序问题
- 没有提供堆外排序内存
- 现在还不稳定

但是我认为Unsafe Shuffle，对于Spark的设计来说，是一个很好的方式。期待Databricks提供性能报告。

这就是本篇文章的全部内容了。还是建议你读一下源码，因为源码真的很有趣。
