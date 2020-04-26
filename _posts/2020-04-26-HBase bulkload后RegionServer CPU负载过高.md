---
layout: post
title: HBase bulkload后RegionServer CPU负载过高
date: 2020-04-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

bulkload时，日志出现了大量CallException。猜测是由于其它进程占用了大量CPU资源，导致HBase进程无法响应。但是登上RegionServer后发现，是HBase进程占用大量CPU资源。

然后用top命令查看是进程内哪个线程的CPU占用过高，用jstack导出线程调用栈后，发现是此线程是在compaction。

经过排查，发现导致这个问题的原因是bulkload数据进HBase时，文件太多导致HBase进行Split和Compaction，进而导致HBase占用CPU负载过高导致不响应后面的请求导致的。

bulkload时会发生Split和Compaction有两种原因：
1. 表中有数据
2. bulkload时文件太多

对应的解决方法为：
1. 每次写这张表的时候先删掉重建
2. 减少任务的分区数量，增加executor内存，从而减少bulkload时文件的数量
3. 增大StoreFile的尺寸。从而减少Region的split
