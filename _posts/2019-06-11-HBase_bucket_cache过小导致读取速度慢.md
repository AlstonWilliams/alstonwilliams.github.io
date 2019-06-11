---
layout: post
title: HBase bucket cache过小导致读取速度慢
date: 2019-06-11
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

关于Bucket Cache的详细介绍请看[HBase Block Cache（块缓存）](https://www.cnblogs.com/zackstang/p/10061379.html)

我们在跑数据的时候，发现读取的速度很慢，原本3分钟就完成的任务，现在需要20分钟。查看HBase日志，发现有大量这种信息:
~~~
2019-06-11 16:17:03,081 WARN  [regionserver/xxx-BucketCacheWriter-1] bucket.BucketCache: Failed allocation for 71468eed39974dfab0e5fc3f9fc84576_52284323; org.apache.hadoop.hbase.io.hfile.bucket.BucketAllocatorException: Allocation too big size=2097861; adjust BucketCache sizes hbase.bucketcache.bucket.sizes to accomodate if size seems reasonable and you want it cached
~~~

经过查阅资料得知，默认情况下，`hbase.bucketcache.bucket.sizes`是64KB。而我们拿到的明显大于这个尺寸，所以导致写不进去，进一步导致速度慢。

而HBase的读流程为，先读MemStore，然后读BlockCache，然后再读BucketCache，如果都找不到，去HFile里面读取到block，然后放到Cache里，并返回结果。

对于我们现在的场景，明显这个流程很多余。我们尝试配置`hbase.bucketcache.bucket.sizes`为3MB，但是读HBase直接会挂掉。后来我们去掉了Bucket Cache以后，任务的时间就缩减到4分钟了。
