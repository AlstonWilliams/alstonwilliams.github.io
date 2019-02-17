---
layout: post
title: HDFS源码解析(5)-replication
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
replication在HDFS中的地位极高，很多地方都用到了它。比如我们前面介绍的lease recovery，以及你通过`hdfs dfs -setrep -R `命令设置的replica数量，等等很多场景。

在这篇文章中，我们会介绍，NameNode如何指示DataNode进行replication。

我们先来看一幅流程图:
![](https://upload-images.jianshu.io/upload_images/4108852-6216fee0186899f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

NameNode中有一个专门的线程，在`BlockManager`中，叫做`ReplicationMonitor`，会检测有没有需要replication的block。

当看到有需要replication的block的时候，它会按照优先级进行replication。

总共有五种优先级。比如说，只有一个replica的block的replication的优先级要比有两个replica的block的replication优先级更高。因为前者更容易丢失数据。具体是哪五种，请自行查看源码。

然后，会选择一个DataNode作为Source，即我们常说的 replication pipeline的Source，来进行replication。

在选择Source时，会优先选择那些处于`DECOMMISSION_INPROGRESS`状态的DataNode，因为通常由于不会给这些节点分配写请求，所以它们的负载更低。

然后，NameNode会选择targets，即replication pipeline中的其他节点，根据我们熟悉的block分配策略。

然后，NameNode会把block replication放入到一个`pending`队列中，这样我们就可以进行失败重试。

然后，通过heartbeat的`BlockCommand.TRANSFER` command来告诉Source开始replication pipeline。

然后DataNode都会顺着这个pipeline发送给下一个DataNode，并且受到下一个DataNode的ACK时，才会给前面的DataNode发送ACK。
