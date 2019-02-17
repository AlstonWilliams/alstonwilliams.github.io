---
layout: post
title: YARN源码解析(9)-ApplicationMaster容错
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在上一篇文章中，我们介绍了TaskAttempt是如何做到容错的。我们可以看到，TaskAttempt的容错很简单，就是ApplicationMaster让ResourceManager重新分配一个Container，就好了。

而除了TaskAttempt需要容错之外，ApplicationMaster更需要容错性。毕竟它相当于MapReduce的大脑。在这篇文章中，我们会详细介绍这个过程。

其实ApplicationMaster的容错也是蛮简单的。

从ApplicationMaster的启动过程中，我们可以看到，它会调用一个叫做**processRecovery()**的方法。

![](https://upload-images.jianshu.io/upload_images/4108852-578ce6e2b47c3db5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

顾名思义，这个方法就是用于故障恢复的。

我们来大体看一下这个方法的实现:

![](https://upload-images.jianshu.io/upload_images/4108852-6597d9bedfd3b42f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，会首先检查我们是否设置了允许ApplicationMaster，以及OutputCommitter是否支持Recovery，最后会检查你设置的Reducer的数量是否大于0，以及Shuffle key是否有效。

我们可以看到，如果你设置的Reduce的数量大于0，并且shuffle key无效，那么是不会进行recovery的。

我们来看看是如何进行recovery的。在**parsePreviousJobHistory()**这个方法中:

![](https://upload-images.jianshu.io/upload_images/4108852-0211dcdc168ec018.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，就是读取JobHistory进行恢复的。

从**TaskImpl**中，我们能够看到，当一个Task执行成功时，就会写入到JobHistory:

![](https://upload-images.jianshu.io/upload_images/4108852-e13dc724639aa108.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
