---
layout: post
title: HDFS源码解析(1)-heartbeat
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
heartbeat是HDFS中最重要的RPC中的一个，DataNode通过heartbeat告诉NameNode的关于DataNode的存活状态，以及DataNode的一些信息，比如，有多少可用存储空间。

然后，NameNode会给DataNode发送一些Command，基于NameNode上为各个NameNode维护的一些信息，比如有哪些block要被删除等。

大致上NameNode和DataNode之间的heartbeat的过程如下:

![](https://upload-images.jianshu.io/upload_images/4108852-a1bd3e5a622eb37b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


DataNode通过Heartbeat发送给NameNode的信息包括这么一些:
- DatanodeRegistration：NameNode用它来验证DataNode
- xceiverCount：此DataNode中用于接受Client请求以及其他DataNode请求的线程的数量
- xmitsInProgress：此DataNode中用于向其他DataNode发送数据的线程的数量
- failedVolumes：此DataNode上失败的Volume的数量
- StorageReport： 各个Storage的信息
- cacheCapacity：此DataNode上可用的Cache的容量
- cacheUsed：此DataNode上已用的Cache的容量

如下图所示:

![](https://upload-images.jianshu.io/upload_images/4108852-8dc69b38ee36a780.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后，NameNode最终会调用`DatanodeManager.handleHeatbeat()`来处理heartbeat。

关于这个方法的处理过程，在上面的流程图中已经写的非常详细了，主要就是做这么四件事：
- 如果这个DataNode没有注册过，那么就回复给DataNode一条`RegisterCommand`提醒它注册
- 如果NameNode此时处于`safe mode`中，那么就发送一条空的Datanode Command
- 如果检测到需要lease recovery，那么就发送一条`BlockRecoveryCommand`
- 否则的话，就把跟这个DataNode对应的要传输block，以及要删除的block，要缓存的以及取消缓存的block等，通过对应的Command发送给DataNode

这里第三点以及第四点都是蛮重要也蛮复杂的部分，我们会在后面的文章中详细介绍。

在DataNode收到NameNode发送过来的Command之后，就会做相应的处理，对应的代码如下:

![](https://upload-images.jianshu.io/upload_images/4108852-c65770bc4225331c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时，在DataNode中，还会根据对应的处理结果，来决定是否进行`block report`以及`cache report`等。

heartbeat的实现，本身并不复杂，复杂的是NameNode收到heartbeat后对应的处理，如`lease recovery`等。
