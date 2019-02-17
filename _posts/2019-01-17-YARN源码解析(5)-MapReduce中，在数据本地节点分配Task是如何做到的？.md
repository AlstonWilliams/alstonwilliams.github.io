---
layout: post
title: YARN源码解析(5)-MapReduce中，在数据本地节点分配Task是如何做到的？
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在前面一篇文章中，我们讲到了ResourceManager,NodeManager以及ApplicationMaster的职责，以及它们的工作流程。

我们提到了，ApplicationMaster会通过向ResourceManager发送ResourceRequest这个数据结构，来获得Container，然后再联系对应的NodeManager进行Container的启动。

我们也提到了，ResourceRequest这个数据结构中，包含了一个属性Resource location，这个属性指定了我们期望在哪个节点分配Container。

没错，正是这里，让MapReduce可以让Task在数据本地节点运行，从而减少集群内部数据的传输，提高性能。

从**RMContainerRequestor**中，我们可以看到，如下代码:

![](http://upload-images.jianshu.io/upload_images/4108852-f9012d69959e383e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-803a18436c26c920.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中**ContainerRequest**中就包含了一个**hosts**这个字段，这个字段的值，就是对应InputSplit对应的hosts.

所以，就这样做到了Task在数据本地节点分配。
