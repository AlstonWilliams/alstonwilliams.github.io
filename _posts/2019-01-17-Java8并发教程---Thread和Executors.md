---
layout: post
title: Java8并发教程---Thread和Executors
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---

本教程分为三个部分,这是第一部分.

在本教程中,我们大量使用了Java8 中的 lambda表达式.如果你对此不是很熟悉,请自行查阅资料来了解. 当然,你也可以看[这篇](http://winterbe.com/posts/2014/03/16/java-8-tutorial/).

## Thread and Runnable

现代操作系统,都支持通过进程和线程来实现并发.进程是程序的运行时的实例.程序是静态的,而进程是动态的.进程与进程之间,相互独立.例如,如果你运行一个Java程序,操作系统就会生成一个和其他进程并行运行的进程.而在进程内部,我们又可以通过使用线程来并发的执行操作,并充分利用现代机器多核的优势.

Java从JDK1.0开始,便开始支持线程了.我们在启动一个线程之前,必须告诉它要它执行什么操作.我们可以通过实现** Runnable**接口来告诉线程它要执行的操作,如下图所示:

![](http://upload-images.jianshu.io/upload_images/4108852-e5fee1f5dc4ccc54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们通过lambda表达式来实现了** Runnable**接口.我们让它做的事是,将线程的名称输出到控制台.在我们启动线程之前,我们先直接启动** Runnable**,看看会发生什么.

结果应该是这样:


![](http://upload-images.jianshu.io/upload_images/4108852-1f368c2624a86ada.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

或者是这样:


![](http://upload-images.jianshu.io/upload_images/4108852-5b3715aa8cd85f28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因为新创建的线程和主线程现在是并行运行,所以我们并不能确定最后的那两行输出的顺序.而且因为这种顺序的不确定性,导致并发编程比单线程编程要困难的多.

我们可以让线程暂停一段时间.我们一般用这来模拟需要执行很长时间的任务的线程.


![](http://upload-images.jianshu.io/upload_images/4108852-3c2c4a52327d34e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如果你运行上面的代码,你会看到,第一条输出和第二条输出中间,间隔了一秒. ** TimeUnit**是一个很有用的枚举,它简化了对时间单元的处理.

使用Thread类来编程,很容易产生错误.所以,Java在2004年发布的JAVA5引入了** Concurrency API**.这些API位于** java.util.concurrent**包中,并且包含很多有用的类.从那之后,这些** Concurrency API**就包含在每个Java版本中了.甚至在Java8中,又为并发引入了新的类和方法.

下面我们来看一下** Concurrency API**中,最重要的那部分- executor services.

## Executors

** Concurrency API**打算用** ExecutorService**来取代** Thread**.Executor能够运行异步任务,并且管理着一个线程池.线程池中的线程都可以被复用.所以我们只需要通过一个** executor service**,就能运行很多的需要并发执行的任务.

下面这个例子演示了如何使用** ExecutorService**:


![](http://upload-images.jianshu.io/upload_images/4108852-500180394ef39d21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** Executors**是一个工厂类,我们可以用它来创建各种各样的** Executor Service**.这里创建了一个只有一个线程的** Executor Service**.

输出的结果很容易理解.你可能会注意到,这个程序一直在运行,一直都不停止.实际上,我们需要手动停掉** Executor**,否则它会一直运行,来监听新的任务.

** ExecutorService**提供了两个方法,让我们停掉** Executor**.一个是** shutdown()**,它会等待当前正在运行的任务停止.另一个是** shutdownNow()**,它会中断全部的正在运行的任务,并让** Executor**立即停止.

我们使用如下图所示的这种方式,来停掉** Executor**更好:


![](http://upload-images.jianshu.io/upload_images/4108852-0511be1c96d9b00f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** Executor**会等待一段时间,让正在运行的任务运行完.超过五秒之后,就停掉全部的正在运行的task.

## Callables和Futures

除了** Runnable**,** Executor**还接受** Callable**作为参数.** Callable**和** Runnable**的区别在于,** Callable**是有返回值的.

下图中的** Callable**,会先暂停一秒钟,然后输出一个整数:


![](http://upload-images.jianshu.io/upload_images/4108852-4b91fe4289c00f84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过** Callable**来创建** Executor**的方式,和通过** Runnable**来创建** Executor**的方式一样.那么我们如何来获取** Callable**的返回值呢?我们可以通过** Future**对象来获取.因为** submit()**方法是非阻塞的,它不会等待任务完成,然后直接返回任务的返回值.而是通过返回一个** Future**对象,其中会封装任务的返回值.等任务完成后,我们就能通过这个** Future**对象,来获取** Callable**的返回值.


![](http://upload-images.jianshu.io/upload_images/4108852-e01d51a3ce785657.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在通过** submit()**方法提交完任务之后,我们通过** isDone()**方法,来查看** Future**对象是否完成.这里当然不会,因为线程会在返回值之前,先暂停一秒.

调用** get()**方法,会阻塞当前线程,直到** callable**执行完毕.现在** Future**对象完成了,我们可以在控制台中,看到如下结果:


![](http://upload-images.jianshu.io/upload_images/4108852-affae718bdf8ca2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** Future**对象和** Executor Service**之间,有轻微的耦合.需要注意的是,如果你在** Future**完成之前,结束** Executor**,那么它会抛出异常.


![](http://upload-images.jianshu.io/upload_images/4108852-a1371e9857c2b954.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


你可能已经注意到了,这里我们是通过** newFixedThreadPool(1)**方法,来创建的** ExecutorService**,它会创建一个只有一个线程的线程池.其等价与** newSingleThreadExecutor**.但是我们可以通过调整其参数来改变该线程池的大小.

## Timeouts

** future.get()**方法,会阻塞当前线程,直到** Callable**执行完毕.那如果**Callable**执行的是一个死循环呢?这会导致我们的程序失去响应.我们可以通过设置超时时间,来解决这个问题:


![](http://upload-images.jianshu.io/upload_images/4108852-45172f9443dbbf5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面的代码会抛出** TimeoutException**:


![](http://upload-images.jianshu.io/upload_images/4108852-cbb4284fe2c597f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


你应该知道为何抛出这个异常:我们设置其最多等待1秒,但是** Callable**执行却需要两秒.

## InvokeAll

** Executor Service**支持通过调用** invokeAll()**方法,来传入多个** Callable**,实现一次执行多个任务的目的.这个方法,其参数是** Callable**的集合,其返回值,是** Future**对象的集合.


![](http://upload-images.jianshu.io/upload_images/4108852-bac26adfa224b111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## InvokeAny

另一个一次执行多个任务的方法是** invokeAny()**方法,这个方法和** invokeAll()**方法,有一些不同.这个方法不会返回** Future**对象,它会一直等到第一个** Callable**运行结束,然后返回其返回值.

我们使用下面的帮助类,来生成** Callable**.


![](http://upload-images.jianshu.io/upload_images/4108852-80133f01b82f4760.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后执行下面的代码,它会返回需要执行的时间最短的任务的返回值:


![](http://upload-images.jianshu.io/upload_images/4108852-0a45b85e7a32d470.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面的代码中,通过** newWorkStealingPool()**来创建了另一种** ExecutorService**.这种** ExecutorService**,其线程池中的线程的数量,默认为我们的机器的核数.

## Scheduled Executors

我们现在已经知道如何来通过** Executor Service**启动线程了.那如果有一个任务,需要重复运行很多次,或者定时执行,那我们该怎么办呢?

我们可以使用** Scheduled Executors**.

下面的代码,会在三秒后,启动一个线程来执行任务:


![](http://upload-images.jianshu.io/upload_images/4108852-c76357afe29a7781.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** schedule()**方法,会返回一个** ScheduleFuture**,相对于普通的** Future**来说,它增加了一个** getDelay**方法,来查看还剩多少时间来启动线程执行任务.

** ScheduleExecutorService**提供了** scheduleAtFixedRate()**和** scheduleWithFixedDelay()**这两个方法.前者会按一定的频率来执行任务.下面的例子会每秒执行一次任务:


![](http://upload-images.jianshu.io/upload_images/4108852-2046d69238a5820b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** scheduleAtFixedRate()**方法指定的间隔,不包括任务执行的时间.所以,如果你让那些需要两秒来执行的任务,每隔一秒执行一次,线程池会很快达到容量上限.

针对上面的那种情况,你应当使用** scheduleWithFixedDelay()**方法.这个方法的参数,是一个任务完成后,再过多久才执行下一个任务.


![](http://upload-images.jianshu.io/upload_images/4108852-d7423f9b9339dd1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 原文地址
http://winterbe.com/posts/2015/04/07/java8-concurrency-tutorial-thread-executor-examples/
