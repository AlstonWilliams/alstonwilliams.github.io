---
layout: post
title: HDFS源码解析(2)-block-report
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
除了heartbeat，block report应该说是HDFS中另一个最重要的RPC。

DataNode通过block report告诉NameNode，它都有哪些block，然后NameNode根据block report来确定一个DataNode上哪些block是无效的，哪些应该被删除，以及哪个block应该被replicate到其他的机器上，以及如何进行replicate。

在这篇文章中，我们会介绍NameNode收到DataNode发送来的block report之后，是如何进行处理的，这一个过程。

我们先来看一个流程图:

![](https://upload-images.jianshu.io/upload_images/4108852-e84c8921db5dc42b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，第一步就是根据NameNode上为这个DataNode维护的一些block的元数据的信息，来确定一个block的状态。

有这么几个场景:
- DataNode报告的block在NameNode中找不到，那么，这个block自然就是无效的。可能是DataNode中的这个block已经被更新的block给替代掉了。因为一个有效的block无论何时，在NameNode中都应该能找到。
- NameNode中存在DataNode中不存在的block。直接在NameNode的元数据中删除就好了
- DataNode报告的block的元数据，和NameNode中的不符，比如，block的长度不同，或者时间戳不对，就要先告诉DataNode把旧的block删除掉，然后告诉其他的DataNode把最新的block replicate到对应的DataNode上
- DataNode报告的block处于`UNDER CONSTRUCTION`状态，那么就把它加到NameNode中维护的关于block的under construction的replicas的列表中
- 如果DataNode报告的block能在NameNode中找到并且这个block是处于FINALIZED状态，那么就把它添加到NameNode维护的关于block的已经完成的replicas的列表中

在NameNode中，一个block可能处于下面的几个状态:
- UNDER_CONSTRUCTION: This is the state when it is being written to. An UNDER_CONSTRUCTION block is the last block of an open file; its length and generation stamp are still mutable, and its data is visible to readers. An UNDER_CONSTRUCTION block in the NameNode keeps track of the write pipelines, and the locations of its RWR replicas.
- UNDER_RECOVERY: If the last block of file is in UNDER_CONSTRUCTION state when the corresponding client's lease expires, then it will be changed to UNDER_RECOVERY state when block recovery starts.
- COMMITTED: COMMITTED means that a block's data and generation stamp are no longer mutable, and there are fewer than the minimal-replication number of DataNodes that have reported FINALIZED replicas of same GS/length. In order to service read requests, a COMMITTED block must keep track of the locations of RBW replicas, the GS and the length of its FINALIZED replicas. An UNDER_CONSTRUCTION block is changed to COMMITTED when the NameNode is asked by the client to add a new block is changed to COMMITTED when the NameNode is asked by the client to add a new block to the file, or to close the file. If the last or the second-to-last blocks are in COMMITTED state, the file cannot be closed and the client has to retry.
- COMPLETE： A COMMITTED block changes to COMPLETE when the NameNode has seen the minimum replication number of FINALIZED replicas of matching GS/length. A file can be closed only when all its blocks become COMPLETE. A block may be forced to the COMPLETE state even if it doesn't have the minimal replication number of replicas, for example, when a client asks for a new block, and the previous block is not yet COMPLETE.

![](https://upload-images.jianshu.io/upload_images/4108852-07b1e7aa781ff153.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在DataNode上，一个replica有这么几种状态:
- FINALIZED: When a replica is in this state, writing to the replica is finished and the data in the replica is "frozen" (the length is finalized), unless the replica is re-opened for append. All finalized replicas of a block with the same generation stamp should have the same data. The GS of a finalized replica may be incremented as a result of recovery.
- RBW(Replica Being Written): This is the state of any replica that is being written, whether the file was created for write or reopen for append. And RBW replica is always the last block of an open file. The data is still being written to the replica and it is not yet finalized. The data of an RBW replica is visible to reader clients. If any failures occur, an attempt will be made to preserve the data in an RBW replica.
- RBR(Replica waiting to be recovered): If a DataNode dies and restarts, all its RBW replica will be changed to the RWR state. An RWR replica will either become outdated and therefore discarded, or will participate in lease recovery.
- RUR(Replica Under Recovery): A non-TEMPORARY replica will be changed to the RUR state when it is participating in lease recovery.
- TEMPORARY: A temporary replica is created for the purpose of block replication. It's similar to an RBW replica, except that its data is invisible to all reader clients. If the block replication fails, a TEMPORARY replica will be deleted.

![](https://upload-images.jianshu.io/upload_images/4108852-04b6ae1e9cc242d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体的代码，都在`BlockManager.processReport()`中，大致的流程，在上面也已经给出来了，请各位自行阅读源码。
