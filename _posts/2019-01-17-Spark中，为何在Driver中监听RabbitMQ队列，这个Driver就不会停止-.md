---
layout: post
title: Spark中，为何在Driver中监听RabbitMQ队列，这个Driver就不会停止-
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
在Driver中，我们有这么一段代码:
~~~
val rabbitMQAccessor = realtimeLabelCompute.createRabbitMQAccessor()
while (true) {
            val message = rabbitMQAccessor.poll()

            realtimeLabelCompute.processMessage(message)

            rabbitMQAccessor.ack()
}
rabbitMQAccessor.close()
~~~

很诡异的是，我们发现，当我们用`yarn -kill`命令kill掉它的时候，它依然不会停止，而是会继续监听这个队列。只不过`SparkContext`确实被关掉了。

我们本来是这样子猜测的，`SparkContext`启动时，会启动一个组件，在一个单独的线程中，当它接收到ApplicationMaster的kill消息时，就kill掉Driver线程。然而，由于Driver线程，在`rabbitMQAccessor.poll()`这里，会有`wait()`操作，所以没有被kill掉。

这种想法实在是无厘头。一个是，子线程怎么会干掉主线程。另一个是，如果能干掉，跟主线程是否`wait()`有什么关系。

好在，在[Mastering Apache Spark](https://jaceklaskowski.gitbooks.io/mastering-apache-spark/yarn/spark-yarn-applicationmaster.html#userClassThread)中找到了答案:

![](https://upload-images.jianshu.io/upload_images/4108852-327081cd1b1b55b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就是说，在YARN这种模式中，当ApplicationMaster运行时，它会把用户代码放到一个单独的线程来运行。然后用**join**方法，等待这个线程的结束。

而我们的用户代码里面，包含了一个`while`循环，而且还有`wait()`，所以基本上不可能结束。这就导致Driver也不会结束。

那为什么即使**yarn -kill**，它都停止不了呢？个人猜测是，即使接收到`kill`命令，它也不会用`System.exit()`这种强制退出的方式。所以，用户代码线程就高枕无忧，依然在运行，导致Driver停不了。
