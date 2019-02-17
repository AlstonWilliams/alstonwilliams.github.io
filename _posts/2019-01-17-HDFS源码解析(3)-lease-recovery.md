---
layout: post
title: HDFS源码解析(3)-lease-recovery
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
Lease recovery指的是，当一个文件超过一个时间段，没有Client和它交互，自动关闭这个文件的过程。

一般出现这种情况是由于Client中间连接失败。

处理lease recovery，需要NameNode和DataNode这两端共同的努力。

我们在前面的heartbeat那节，说明了NameNode会在处理heartbeat的时候，如果需要lease recovery，就给DataNode发送RELEASE_RECOVERY.

那么，DataNode在收到这条Command之后，它要如何处理呢？

我们来看这么一幅流程图:

![](https://upload-images.jianshu.io/upload_images/4108852-47f6a4b25cd34e59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，DataNode做的主要工作，就是从其他的包含这个block的DataNode上接收关于这个block的一些信息，并筛选出来处于最佳状态的那一个。

这里说的`这个block`，指的是这个文件的最后一个block。因为进行release recovery时，只可能这个文件的最后一个block不一致。

找到处于最佳状态的block之后，就通知NameNode。然后NameNode做如下处理：

![](https://upload-images.jianshu.io/upload_images/4108852-6257cb3e8384ca71.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，NameNode做的工作，就是检查这个block是否需要被replication，如果需要的话，就进行replication。

并且移除这个lease。

这样就可以让这个文件保持一致性，并且数据总是最新的。

我们可以看到，这里并没有通知DataNode移除它上面的旧的错误的replica，而只是进行replication。那么，DataNode怎么知道要移除哪些旧的replicas呢？

如果你读过前一篇，关于block report的那一篇，相信你这里已经有答案了。不过我们还是会放到下一篇中再介绍一下这个问题。

## 其他资料

[Understanding HDFS Recovery Processes (Part 1) ](http://blog.cloudera.com/blog/2015/02/understanding-hdfs-recovery-processes-part-1/)
[Understanding HDFS Recovery Processes (Part 2)](http://blog.cloudera.com/blog/2015/03/understanding-hdfs-recovery-processes-part-2/)

注意，这两篇文章中，有一些外链指向了一些HDFS的设计文档，你应当下载下来查看一下，再结合源码阅读，会让你对Lease recovery更加清晰.
