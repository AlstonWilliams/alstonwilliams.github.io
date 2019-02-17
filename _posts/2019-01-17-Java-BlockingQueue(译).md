---
layout: post
title: Java-BlockingQueue(译)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
在**java.util.concurrent**包中的**BlockingQueue**接口,其存取操作,是线程安全的.在这篇文章中,我会告诉你,如何来使用这个接口.

这篇文章中,不会告诉你如何实现一个**BlockingQueue**.如果你对此感兴趣,那请参考文末的资料.

## BlockingQueue的用法

**BlockingQueue**的典型使用场景是有一个生产者生产产品,然后有一个消费者消费产品.如下图所示:

![](http://upload-images.jianshu.io/upload_images/4108852-b996a0578ac7ef4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

生产者线程会生产一个产品,然后把它放入队列中,直到这个队列达到了上限.一旦队列达到上限,生产者线程会被阻塞,直到有一个消费者线程消费了队列中的一个产品.

同样,消费者线程会从队列中取出一个产品,然后消费它,直到这个队列中没有产品了.如果队列中没有产品,那么消费者线程会被阻塞,直到队列中有产品.

## BlockingQueue中的方法

针对队列中产品的操作,**BlockingQueue**提供了四类方法:


![](http://upload-images.jianshu.io/upload_images/4108852-4e4657d209fdc3aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们来解释一下第一行中的那些字段的含义:

- Throws Exception:
	如果操作不能立即执行,比如消费者线程尝试消费空队列中的产品,则会抛出异常.
- Special Value:
	如果操作不能立即执行,则返回一个特殊值.一般是true或者false.
- Blocks:
	如果操作不能立即执行,则阻塞此方法.
- Times Out:
	如果操作不能理解成功,则阻塞线程并等待给定的时间.返回一个Special Value来表明调用是否成功.
    
队列中,不可以插入一个null.如果你这么做,那么**BlockingQueue**会给你抛出一个**NullPointerException**.

我们可以访问**BlockingQueue**中的任意产品,而不用非得是开头或者结尾位置的产品.例如,如果你打算移除队列中的一个产品,那你可以采用**remove(0)**的方法.然而,这个方法的效率并不是特别高,所以你应该尽量避免使用它.

## BlockingQueue的实现
**BlockingQueue**只是一个接口,我们实际上会使用的,还是它的实现.**java.util.concurrent**包中,提供了这些**BlockingQueue**的实现:

- [ArrayBlockingQueue](http://tutorials.jenkov.com/java-util-concurrent/arrayblockingqueue.html)
- [DelayQueue](http://tutorials.jenkov.com/java-util-concurrent/delayqueue.html)
- [LinkedBlockingQueue](http://tutorials.jenkov.com/java-util-concurrent/linkedblockingqueue.html)
- [PriorityBlockingQueue](http://tutorials.jenkov.com/java-util-concurrent/priorityblockingqueue.html)
- [SynchronousQueue](http://tutorials.jenkov.com/java-util-concurrent/synchronousqueue.html)

我们会在后面的文章中,介绍它们.

## BlockingQueue的例子
下面是一个使用**BlockingQueue**的例子.其中我们用到了**ArrayBlockingQueue**这个实现.


![](http://upload-images.jianshu.io/upload_images/4108852-b92cbea9ff410c8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-6f99303b02c45f01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-3e3332270b65f8bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


消费者每次生产完一个产品,都会暂停一秒中,这将导致我们的消费者也暂时被阻塞.

## 原文链接
[原文](http://tutorials.jenkov.com/java-util-concurrent/blockingqueue.html)

## 参考资料
[实现BlockingQueue](http://tutorials.jenkov.com/java-concurrency/blocking-queues.html)
