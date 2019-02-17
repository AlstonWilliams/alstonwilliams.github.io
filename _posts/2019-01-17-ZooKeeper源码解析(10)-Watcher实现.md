---
layout: post
title: ZooKeeper源码解析(10)-Watcher实现
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
我们知道，ZooKeeper有一个非常重要的功能，就是做分布式锁．而这个分布式锁，就是通过ZooKeeper的**Watcher**来实现的．

在这篇文章中，我们将会介绍，在ZooKeeper中，**Watcher**到底是如何实现的．

## ZooKeeper Watch介绍

在ZooKeeper中，有三个读操作:**getData(), getChildren(), exists()**，都接受一个叫做**Watch**参数．

在ZooKeeper中，有两种**Watch**，分别是**data watches**和**child watches**，**getData()和exists()**会设置**data watches**，而**getChildren()**会设置**child watches**．

**Watch**是由服务器端维护的一个集合，它会为每个客户端维护这么一个集合．

**setData()**会触发对应的znode上的**data watches**，**create()**会触发对应znode的**data watches**以及它的父节点的**child watches**，**delete()**会触发对应znode的**data watches以及child watches**，以及其父节点的**child watches**．

**Watches**只会被触发一次．

更多关于**Watch**的介绍，请到[这里](https://zookeeper.apache.org/doc/trunk/zookeeperProgrammers.html#ch_zkWatches)

## ZooKeeper中Watch的实现

在**ZooKeeper**中，有一个**Watcher**接口：

![](http://upload-images.jianshu.io/upload_images/4108852-b6e4a955fe3f5280.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个**Watcher**接口中，定义了一个方法:**process(WatchedEvent event)**．

就是这个方法，定义了当事件被触发时，要进行的操作．

而实现**Watcher**接口的类，有三个，都是我们很熟悉的：

- ServerCnxn
- NIOServerCnxn
- NettyServerCnxn

我们这里主要看一下**NIOServerCnxn**中的实现：

![](http://upload-images.jianshu.io/upload_images/4108852-a74ab303df2b8bfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，其**process(WatchEvent event)**方法，就是将**WatchEvent**发送给客户端．

## ZooKeeper中Watch的原理

在之前的文章中，我们介绍请求处理的时候，提到了有一个生产链似的请求处理方式，其中最后的一个**RequestProcessor**就是**FinalRequestProcessor**，而正是在这个**FinalRequestProcessor**中，设置了Watch．

我们来看一下**FinalRequestProcessor**中的**processRequest(Request request)**部分的实现．

从这个方法中，我们可以这个一段代码块：

![](http://upload-images.jianshu.io/upload_images/4108852-170fbbe987b1fb89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，其中调用了**ZKDatabase**的**getData()**方法来设置获取数据．并且最后有一个关于**Watch**的参数．

而我们上面正好介绍过，在使用**getData()**时，可以设置**Watch**．

我们跟进去，可以发现它最后就是调用了**DataTree**的**getData()**方法．

![](http://upload-images.jianshu.io/upload_images/4108852-e35f66f97880d09d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，这儿只不过是记录了一下这个**Watch**．

这里读者们可能就有一个疑问，就是，之前我们说过，ZooKeeper为每一个客户端维护一个Watch集合，但是这里我们只看到直接就放到了**DataTree**这一个共享的数据结构里了呀．

这是为什么呢？

因为**Watch**的实现**NIOServerCnxn**天生就是客户端独立的呀．

那**Watch**是怎么被消费掉，怎么被触发的呢？

我们回到**FinalRequestProcessor**的**processRequest()**方法，可以看到这么一段代码：

![](http://upload-images.jianshu.io/upload_images/4108852-2d6fdb3adc4d6739.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，最重要的就是图中我们圈出来的那一行．

我们看一下在**zks.processTxn()**的实现．

![](http://upload-images.jianshu.io/upload_images/4108852-68ad3a907319df87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中这个方法中，最重要的还是我们圈出来的那一行．

我们再跟进去，发现它调用了**DataTree**的**processTxn()**方法．我们直接来看这个方法的实现．

![](http://upload-images.jianshu.io/upload_images/4108852-cf7302ab0745e4ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们只分析**OpCode.create**时的情况.

我们看到，就是调用了**createNode()**方法．

在这个方法的尾部，我们可以看到，就触发了**dataWatches**和**childWatches**．

![](http://upload-images.jianshu.io/upload_images/4108852-615373e665906e49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就这样完成了**Watch**的触发．
