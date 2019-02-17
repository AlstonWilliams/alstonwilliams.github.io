---
layout: post
title: ZooKeeper源码解析(9)-几种RequestProcessor
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
在[ZooKeeper源码解析(7)-请求处理(上)](http://www.jianshu.com/p/f8a218857474)和[ZooKeeper源码解析(8)-请求处理(下)](http://www.jianshu.com/p/0b3f7b2066e8)中，我们已经介绍过了，**ZooKeeperServer**主要就是通过**RequestProcessor**来进行生产链似的处理请求．

那么，在这篇文章中，我们就来介绍一下这几种**RequestProcessor**.

## 几种RequestProcessor的实现

在ZooKeeper中，主要有下面几种**RequestProcessor**：

- AckRequestProcessor
- CommitProcessor
- FinalRequestProcessor
- FollowerRequestProcessor
- ObserverRequestProcessor
- PrepRequestProcessor
- ProposalRequestProcessor
- ReadOnlyRequestProcessor
- SendAckRequestProcessor
- SyncRequestProcessor
- ToBeAppliedRequestProcessor
- UnimplementedRequestProcessor

这些**RequestProcessor**都实现了**RequestProcessor**接口．

我们来看一下**RequestProcessor**这个接口．

![](http://upload-images.jianshu.io/upload_images/4108852-a58ba267fc5d1263.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中处理请求的就是**processRequest(Request request)**这个方法．

## PreRequestProcessor

**PreRequestProcessor**一般是放在处理链的起始部分的，它对请求做一些预处理，比如：

- 检查Session
- 检查要操作的节点及其父节点是否存在
- 检查客户端是否有权限

**PreRequestProcessor**继承自**ZooKeeperCritialThread**，它的**processRequest(Request request)**方法只是将**request**加到**submittedRequests**中．

在它的**run()**方法中，会不断从**submittedRequests**中的内容并进行处理．

![](http://upload-images.jianshu.io/upload_images/4108852-cfab844e47274f58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ProposalRequestProcessor

**ProposalRequestProcessor**用于向**Follower**发送**Proposal**，来完成Zab算法．

![](http://upload-images.jianshu.io/upload_images/4108852-882577d4cc7887b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

正如我们在上面代码中注释写的那样，有一部分Request是不需要进行提议的，比如，来自Follower的请求转发的Request．

## SyncRequestProcessor

**SyncRequestProcessor**用于将请求持久化到磁盘中．

![](http://upload-images.jianshu.io/upload_images/4108852-68c1a2e9dcb206f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## AckRequestProcessor

**AckRequestProcessor**实现的功能特别简单，就是在同意Leader的Proposal之后，给Leader回复一个ACK.

![](http://upload-images.jianshu.io/upload_images/4108852-88a4a8a9eadb7da8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## CommitProcessor

这个**CommitProcessor**用于将来到的**commit request**和本地已提交的**request**进行对比．

![](http://upload-images.jianshu.io/upload_images/4108852-a0189ba4f8344c5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## FinalRequestProcessor

**FinalRequestProcessor**负责将Transaction Log应用到DataTree上，并给客户端回复．

![](http://upload-images.jianshu.io/upload_images/4108852-07dd71ad499369b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 其他RequestProcessor

其他的繁杂的RequestProcessor，我们这里就不介绍了，大多数都是实现很简单的功能．比如，**FollowerRequestProcessor**实现将请求从Follower转发到Leader．

## 总结

这篇文章中，其实对各种RequestProcessor的实现并没有进行深入的解析，而只是大体介绍了一下它们的功能．其实大多数都很简单，只有那个**ProposalRequestProcessor**比较复杂．

在ZooKeeper中，对每个请求，都是通过这种生产链的方式，一步步地进行处理．感觉这样特别美观．
