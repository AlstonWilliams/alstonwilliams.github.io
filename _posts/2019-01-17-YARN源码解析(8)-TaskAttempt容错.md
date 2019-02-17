---
layout: post
title: YARN源码解析(8)-TaskAttempt容错
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
我们前面已经介绍了，一个Application,一个Job正常执行的工作流程。

但是，我们都知道，在一个分布式系统中，容错是很重要的。容错性，也是在设计一个分布式系统的时候，必须要仔细考虑的问题。

了解MapReduce的朋友都知道，在MapReduce执行的时候，每个Mapper或者Reducer都是一个Task。而这些Task又有对应的TaskAttempt，来做到对每个Task的容错性。TaskAttempt指的是，每次Task尝试执行。每个TaskAttempt都有自己的ID。我们也可以指定一个Task最多attempt几次，才会放弃。

那么，在这篇文章中，我们会介绍，ApplicationMaster是如何检测以及处理TaskAttempt出现故障的。

## TaskAttempt故障检测

ApplicationMaster判断一个TaskAttempt是否出现的标准很简单，有这么三点:
- TaskAttempt在正常的过期时间段内，并没有读数据
- TaskAttempt在正常的过期时间段内，并没有写数据
- TaskAttempt在正常的过期时间段内，并没有告诉ApplicationMaster它进入了其他的Phase。

这里的**Phase**指的是什么呢？我们都知道，在Mapper端，会有一个Sort的过程，在Reducer端，会有一个Shuffle的过程，以及对接收到的数据进行Sort的过程。这么，这些过程，不管是*Sort*也好，*Shuffle*也好，就是一个个的Phase。

那么，ApplicationMaster怎么知道有没有上面的三个条件发生呢？

很简单，在ApplicationMaster上，有一个专门的**TaskHeartbeatHandler**，专门用于检测TaskAttempt是否过期。

![](https://upload-images.jianshu.io/upload_images/4108852-e18f4786ecfd727f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的代码中，我们可以看到，就是循环检测超时时间段内，TaskAttemptId是否超时。

还有另一个类，**TaskAttemptListenerImpl**，用于接收NodeManager上的**TaskAttempt**发送来的状态信息。每当**TaskAttemptListenerImpl**收到这些信息，就会更新**TaskHeartbeatHandler**中对应的**TaskAttempt**的**ReportTime**。

![](https://upload-images.jianshu.io/upload_images/4108852-96263c066359b8cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**TaskAttempt**会在切换Phase的时候，直接告诉**TaskAttemptListenerImpl**，""我的状态更新啦。"

![](https://upload-images.jianshu.io/upload_images/4108852-9d6a8d326f4b2b2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中**statusUpdate(umbilical)**方法内部，调用的就是上面我们看到的**TaskAttemptListenerImpl**的**statusUpdate()**方法。

那剩下的两个条件呢？即**TaskAttempt**会告诉**ApplicationMaster**它在过期时间内，读过或者写过数据呢？

为了实现这个功能，在**MapTask**和**ReduceTask**的父类**Task**中，有一个**TaskReporter**.这是一个线程，它的作用就是，当各种Counter更新时，就告诉**TaskAttemptListenerImpl**，"我的状态更新啦"。

![](https://upload-images.jianshu.io/upload_images/4108852-e7b84b6ed8a8bf34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这**各种Counter**，都是啥呢？都是MapReduce内部使用的Counter啦，就是那些报告给你，Mapper多了多少数据，Shuffle了多少数据等，那些Counter。只要你写过MapReduce程序，对它一定不陌生。

而这些Counter都是什么时候更新呢？

当然是在TaskAttempt读写数据时更新啦。

这里拿**MapTask**举例，从**MapTask**中内部的RecordReader的wrapper-**NewTrackingRecordReader**来看，当它调用**nextKeyValue()**方法的时候，就会更新这些Counter。

![](https://upload-images.jianshu.io/upload_images/4108852-a73c5caee3a03d35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而这个方法，会在我们执行Mapper的时候被调用到。

![](https://upload-images.jianshu.io/upload_images/4108852-bccc648c9899563f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至于输出的话，记得之前看过是在**OutputCommitter**中更新的。

## ApplicationMaster处理TaskAttempt故障

顺着**TaskHeartbeatHandler**的处理逻辑往下走，最终，你会发现，ApplicationMaster会做这样的处理:
- 创建一个TaskAttempt
- 跟ResourceManager请求资源
- 重新在NodeManager上调度

这里我们只介绍第一步，第二三步跟正常调度一个Container相同。

![](https://upload-images.jianshu.io/upload_images/4108852-95da51cc49de0b38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，如果没有失败的TaskAttempt，那么就直接发出**TaskAttemptEventType.TA_SCHEDULE**事件进行调度，而如果有，那么就发出**TaskAttemptEventType.TA_RESCHEDULED**事件重新进行调度。
