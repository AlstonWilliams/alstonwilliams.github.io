---
layout: post
title: SparkSQL中distinct vs group by
date: 2019-04-29
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

前两天，同事对我的代码进行CodeReview的时候,看到我写了一句**select distinct(session) from insight.labeled_id where date='20190101' and id ='1'**，告诉我不能这么写，应该写成**select session from insight.labeled_id where date='20190101' and id ='1' group by session**

然后我就有点懵，这两个有啥区别?

同事告诉我，当然有区别，前者相当于Spark中的`groupByKey`，而后者相当于`reduceByKey`。本着怀疑态度，查看了一下这两条SQL的执行计划。

我们先看第一条:
~~~
sql("select distinct(session) from insight.labeled_id where date='20190101' and id ='1'").explain()

== Physical Plan ==
*HashAggregate(keys=[session#198], functions=[])
+- Exchange hashpartitioning(session#198, 200)
   +- *HashAggregate(keys=[session#198], functions=[])
      +- *Project [session#198]
         +- *Filter (isnotnull(id#199) && (id#199 = 1))
            +- *FileScan parquet insight.labeled_id[session#198,id#199,tp#195,date#196] Batched: true, Format: Parquet, Location: PrunedInMemoryFileIndex[hdfs://..., PartitionCount: 1, PartitionFilters: [isnotnull(date#196), (date#196 = 20190101)], PushedFilters: [IsNotNull(id), EqualTo(id,1)], ReadSchema: struct<session:string,id:string>
~~~

我们可以看到，它是先在每个Executor内部做一遍聚合，然后再进行repartition,然后再做一遍全局聚合，其实就跟Spark中的`reduceByKey`是一样的。

我们再来看另一条:
~~~
sql("select session from insight.labeled_id where date='20190101' and id ='1' group by session").explain()

== Physical Plan ==
*HashAggregate(keys=[session#226], functions=[])
+- Exchange hashpartitioning(session#226, 200)
   +- *HashAggregate(keys=[session#226], functions=[])
      +- *Project [session#226]
         +- *Filter (isnotnull(id#227) && (id#227 = 1))
            +- *FileScan parquet insight.labeled_id[session#226,id#227,tp#223,date#224] Batched: true, Format: Parquet, Location: PrunedInMemoryFileIndex[hdfs://..., PartitionCount: 1, PartitionFilters: [isnotnull(date#224), (date#224 = 20190101)], PushedFilters: [IsNotNull(id), EqualTo(id,1)], ReadSchema: struct<session:string,id:string>
~~~

我们可以看到，完全跟第一条一样。

所以其实在SparkSQL中，这两种写法是等价的。第一种会自动被转化成第二种。

但是，在Google的过程中，发现对于某些数据库来说，这两条并不是完全等价的。这一点需要注意。
