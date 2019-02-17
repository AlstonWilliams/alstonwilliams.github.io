---
layout: post
title: Java-NIO核心组件-Selector和Channel
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
昨天我们介绍了一下**SelectorProvider**和**IO multiplexing**.特别是**IO multiplexing**中的**epoll**系统调用,是Linux版本的Java的NIO的核心实现.

那今天我们就来介绍一下, Java NIO中的核心组件, Selector和Channel.这两个组件,对于熟悉Java OIO,而不熟悉Java NIO的朋友来说,理解其作用是极其不易的.

## 前提条件

在阅读这篇文章之前,如果各位不熟悉甚至没有听说过**IO multiplexing**中的**epoll**,请务必花时间去了解一下.了解了它们之后,就很容易理解Java NIO的实现了.

这里我们讲解的是Linux版本且内核版本大于2.6的Java NIO的实现,对于其他的系统或者内核版本较低的Java NIO,其具体实现是不一样的.

举例来说, Linux 内核版本大于等于2.6的Java NIO是采用**epoll**来实现的.而Linux内核版本小于2.6的Java NIO,则是采用**poll**来实现的.

## Selector和Channel

**Selector**和**Channel**的关系,如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-dc7af779b77dd17d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


各位如果了解过**epoll**的话,应该知道**epoll_create**操作会创建一个需要被监听的**file descriptor**.然后,**epoll_ctl**操作会为告诉内核,需要监听一个**file descripitor**的什么事件.最后,使用**epoll_wait**来告诉内核开始监听.

这里我们就可以把**Selector**比作**epoll**中的内核.把**Channel**比作**epoll_create**操作创建的**file descriptor**.这样就很容易理解了吧.

因为Java NIO实际上是给我们对**IO multiplexing**进行了封装,隐藏了其底层的实现.所以我们完全可以这样来理解.

贴出在Java NIO tutorial中看到的一个图片,

![](http://upload-images.jianshu.io/upload_images/4108852-69f97609aad66056.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于这张图片,我实在是不能苟同其说法.我们可以看到,在这张图片中,我们可以看到,一个线程中只有一个**Selector**,每个**Selector**负责监控三个**Channel**.而实际上,一个线程中,并不是必须只能有一个**Selector**.一个**Selector**也不是只能注册三个**Channel**.

在**AbstractSelector**的源码中,我们可以看到,实际上它只维护了一个不再监听的**Channel**的集合:


![](http://upload-images.jianshu.io/upload_images/4108852-159fa90e67130c7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们查看具体的**Selector**的父类,**SelectorImpl**,中的**register**方法的实现.跟具体的**Selector**实现相关的类,在JDK提供的**src.zip**源码包中是找不到的.这里使用**CFR**反编译器反编译**rt.jar**包.从中找到其实现.


![](http://upload-images.jianshu.io/upload_images/4108852-a32be7546bdcb857.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到,它会把**Channel**进一步封装成**SelectionKeyImpl**.然后使用**implRegister**方法来实现具体的注册过程.从**SelectorImpl**的源码中,我们同样可以看到,**implRegister**方法是一个抽象方法,需要其子类来实现具体的注册过程.

这里我们感兴趣的子类是**EPollSelectorImpl**,我们查看其源码,可以看到其中维护了一个从**file descriptor**到**SelectionKeyImpl**的Map.我们刚刚也提到了,**SelectionKeyImpl**中,包装了一个**Channel**,


![](http://upload-images.jianshu.io/upload_images/4108852-fdd2c8b2d0f747eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们从**EPollSelectorImpl**的**implRegister**方法中,也没有看到会对**Map<Integer, SelectionKeyImpl>**这个表示**EPollSelectorImpl**维护的**Channel**的Map进行尺寸限制的操作.即并没有限制一个**EPollSelectorImpl**可以注册的**Channel**的数量.

![](http://upload-images.jianshu.io/upload_images/4108852-8a8539c62d7c9b3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


反而是在**Channel**中,维护了它向**Selector**注册时,**Selector**给其返回的**SelectionKey**的集合.相当于维护了它已经注册的**Selector**的集合.


![](http://upload-images.jianshu.io/upload_images/4108852-2644388dc17c1f04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们查看**AbstractSelectableChannel**的向**Selector**注册的源码:


![](http://upload-images.jianshu.io/upload_images/4108852-6eff17db4fdacc7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到,它会把**Selector**给它返回的**SelectionKey**加入到上面我们说过的那个集合中,我们看看**addKey()**方法的具体实现:


![](http://upload-images.jianshu.io/upload_images/4108852-3362d435b267cc27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在这里我们就可以看到,默认情况下,**Channel**会创建一个容量为3的表示它注册的**Selector**的集合.当它需要向更多的**Selector**注册时,则对这个集合进行扩容.

而并没有提到一个**Selector**中最多可以注册多少个**Channel**.
