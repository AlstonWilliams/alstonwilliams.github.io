---
layout: post
title: Spark调用repartition()方法后相同的key没有被分到一起
date: 2019-04-19
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近在项目中，遇到过好几个问题，调试下来发现是中间我们为了加大partition的数量，调用了`repartition()`方法，但是调用完之后呢，原本相同key的数据处于同一个partition上，后来却不在一个partition上了。

感觉特别奇怪。于是读了一下`repartition()`方法的源码，发现它是将每个partition中的数据，随机的分配到一个新的分区，而不会在乎是否会将相同的key的数据分到同样的分区。我们看它的代码:
~~~
def coalesce(numPartitions: Int, shuffle: Boolean = false,
               partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
              (implicit ord: Ordering[T] = null)
      : RDD[T] = withScope {
    require(numPartitions > 0, s"Number of partitions ($numPartitions) must be positive.")
    if (shuffle) {
      /** Distributes elements evenly across output partitions, starting from a random partition. */
      val distributePartition = (index: Int, items: Iterator[T]) => {
        var position = (new Random(index)).nextInt(numPartitions)
        items.map { t =>
          // Note that the hash code of the key will just be the key itself. The HashPartitioner
          // will mod it with the number of total partitions.
          position = position + 1
          (position, t)
        }
      } : Iterator[(Int, T)]

      // include a shuffle step so that our upstream tasks are still distributed
      new CoalescedRDD(
        new ShuffledRDD[Int, T, T](mapPartitionsWithIndex(distributePartition),
        new HashPartitioner(numPartitions)),
        numPartitions,
        partitionCoalescer).values
    } else {
      new CoalescedRDD(this, numPartitions, partitionCoalescer)
    }
  }
~~~

`repartition()`方法内部就是调用的`coalesce()`方法，我们可以看到，它就是将每个partition中的数据，随机的分配到一个新的分区而已。

所以以后需要注意的是，如果想要重新分区，并且需要将相同的key分到同一个partition上，那么是不能调用`repartition()`或者`coalesce()`方法的。

我们上测试代码:
~~~

import org.apache.spark.{SparkConf, SparkContext}

object TestSparkRepartition {

    def main(args: Array[String]): Unit = {

        val sparkConf = new SparkConf()
            .setMaster("local")
            .setAppName("TestSparkRepartition")

        val datas = List[(String, String)] (
            "1" -> "11", "1" -> "11", "1" -> "11",
            "1" -> "11", "1" -> "11", "1" -> "11",
            "1" -> "11")

        val sparkContext = new SparkContext(sparkConf)
        val rdd = sparkContext.parallelize(datas.toSeq)
        println("------ partitionNumberBefore: " + rdd.getNumPartitions)
        rdd.mapPartitions {
            items: Iterator[(String, String)] => {
                items.foreach(println(_))
                items
            }
        }.collect()

        val afterMorePartitionRDD = rdd.repartition(100).map(item => item._2)
        println("------ partitionNumberAfterMore: " + afterMorePartitionRDD.getNumPartitions)
        afterMorePartitionRDD.mapPartitions {
            items: Iterator[String] => {
                items.foreach(println(_))
                items
            }
        }.collect()

        val afterLessPartitionRDD = rdd.repartition(100).repartition(3).map(item => item._2)
        println("------ partitionNumberAfterLess: " + afterLessPartitionRDD.getNumPartitions)
        afterLessPartitionRDD.mapPartitions {
            items: Iterator[String] => {
                items.foreach(println(_))
                items
            }
        }.collect()

    }

}
~~~
