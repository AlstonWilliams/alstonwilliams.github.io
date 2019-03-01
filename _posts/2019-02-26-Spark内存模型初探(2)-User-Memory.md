---
layout: post
title: Spark内存模型初探(2)-User Memory
date: 2019-02-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
在上一篇文章[Spark内存模型初探(1)-Storage/Execution Memory的使用](https://www.jianshu.com/p/115ebaf525eb)中，我们初步解析了一下Storage/Execution Memory的使用。最后我们也留下了几个问题，等待我们解答。这些问题更多的集中在User Memory上。

而这段时间，经过思索，探索，以及重新阅读[Spark Memory Management](https://0x0fff.com/spark-memory-management/)，终于解决了大部分。

但是，本文其实更多的是作者的猜测，以及配合[Spark Memory Management](https://0x0fff.com/spark-memory-management/)进行验证。并没有一个非常严谨，科学的方法来验证本文的全部猜测。所以希望读者在读本文的时候，保持怀疑的目光。

#### UserMemory是怎样被使用的?当我们在RDD.map(function)中的function中初始化新的对象时，是在哪部分内存被初始化的？是不是就是UserMemory?

这个问题的答案，其实在原文里就已经有了。但是由于当时读那篇文章的时候，对Spark内存并不了解，所以这里并没有读懂。

原文中有这么一段:
>  This is the memory pool that remains after the allocation of Spark Memory, and it is completely up to you to use it in a way you like. You can store your own data structures there that would be used in RDD transformations. For example, you can rewrite Spark aggregation by using mapPartitions transformation maintaining hash table for this aggregation to run, which would consume so called User Memory

要说验证么，我实在是没想出来办法验证。因为从Spark代码中，看不到任何跟User Memory相关的内容。而我们知道，Spark内存分为三部分:Reserved Memory, User Memory, Spark Memory(Storage/Execution Memory)。我们在上篇文章也测试了，`function`中初始化新的对象时，是不会在Spark Memory中分配的，更不会在Reserved Memory，所以可能的地方就只有在User Memory了。

#### 如果我在`function`里就是要初始化很多个对象，超过了这个User Memory的大小的话，Spark会怎样做?

先看原文:
>  its completely up to you what would be stored in this RAM and how, Spark makes completely no accounting on what you do there and whether you respect this boundary or not. Not respecting this boundary in your code might cause OOM error.

我简单写了一点测试代码，来测试这个问题:
~~~
package com.hypers.spark

import org.apache.spark.{SparkConf, SparkContext}


/**
  *
  * Reading:
  * Task max available memory is executor heap.
  * 
  * And this program will cause OOM because:
  *     User Memory just 200M, but we allocate an Array[Byte] whose size is 500MB
  *
  **/
object TestTaskAvailableMemory {

    def main(args: Array[String]): Unit = {
        val sparkConf = new SparkConf()
            .setMaster("spark://localhost:7077")
            .setAppName("TestSparkCanRun")

        val sparkContext = new SparkContext(sparkConf)

        val rdd = sparkContext.parallelize(Seq(1))

        rdd.map {
            item => {
                val line = Array.fill[Byte](489354548)(0)
                (item, line)
            }
        }.repartition(1).foreach(println(_))

    }

}
~~~

这是会导致OOM的。原因在代码的注释里已经写了。

另外，超过User Memory并不一定会导致OOM。现在User Memory的大小大概是200MB左右，我分配一个300MB的Array[Byte]，是没问题的。

我使用的JVM参数是`-Xmx1g -XX:+UseSerialGC -XX:-UseAdaptiveSizePolicy -XX:PretenureSizeThreshold=10000000`。可以看到，超过10M的对象，就要直接在老年代分配了。可我测试时，老年代的大小是600MB左右。已用100MB。所以这个老年代没有足够的空间来进行分配。所以会出现OOM。

为什么不直接在新生代测试呢？因为新生代的大小共有300MB左右，Eden:Survivor1:Survivor2=240MB:30MB:30MB。而我们的User Memory就已经200MB+了，达不到测试的目的。

从这儿我们也能看到，其实Spark中，Storage/Execution Memory的大小，不会被User Memory挤压。从Spark的源代码中，我们就能看到，它只是一个数字，是`MemoryManager`(具体类是`UnifiedUserMemory`或者`StaticUserMemory`)初始化时就提供的，它并不会在运行时动态获取Java Heap的可用内存大小，进而自动伸缩。

另外，Storage/Exeuction Memory也不是说，你Java Heap在我Spark启动的时候，就给我把Storage/Execution Memory这些内存分配好了，只有我能用，其它人不能用。它只是申明，我需要多少Storage/Execution Memory，你User Memory要是用多了，我Storage/Execution Memory到时候用的时候，你要是给我腾不出来，那咱俩同归于尽。

#### Shuffle时，Reducer端接收到的数据，是在哪个部分分配的?是不是就是UserMemory？

我认为是在User Memory分配的。但是由于Shuffle是`ExternalShuffle`，所以并不会占用过多的内存，导致User Memory过大出现OOM。

#### Spark的内存模型，跟Java内存模型之间，是什么关系?

假设我们的User Memory是300MB，而新生代是200MB，并且没有启用超过阈值就自动在老年代进行分配的机制。

那我们如果在User Memory中分配一个250MB的对象，在这种情况下，新生代根本就放不下这个对象，所以即使我们看到User Memory有300MB可用，实际上也不能分配超过200MB的大对象。

其它的Storage/Execution Memory，跟这个道理类似。

## 疑问

我们可以看到，由于Java内存模型的缘故，我们可能无形之间碰到一些莫名其妙的坑。

而Spark又支持off-heap Memory，以及Tungsten，那使用这些又如何呢？有什么优缺点呢？
