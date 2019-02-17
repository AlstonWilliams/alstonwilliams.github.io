---
layout: post
title: ZooKeeper源码解析(3)-Cluster启动过程解析
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
ZooKeeper启动时，有两种模式，第一种是单例模式，这也是默认模式，第二种是cluster模式．今天我们就来探究Cluster模式下，ZooKeeper模式是如何启动的．

## 启动过程

Cluster模式下的入口为**org.apache.zookeeper.server.quorum**．

那么，Cluster模式下，节点是如何一步步启动起来的呢？


![](http://upload-images.jianshu.io/upload_images/4108852-6d364b9aad8dc17a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的这幅图片中，我们可以看到，就是调用的**QuorumPeerMain**的**initializeAndRun**方法．

我们看一下这个方法的实现．

![](http://upload-images.jianshu.io/upload_images/4108852-af174888e7a9ceff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里也很容易理解，就是做了这么几件事：

- 解析配置文件
- 启动一个定时器来清理旧数据并且生成快照
- 如果配置文件中并没有指定**server**，那么就以单例模式启动

如果配置文件没有错误，就调用**runFromConfig()**方法．

在这个方法中，会创建一个**ServerCnxnFactory**类的子类的实例，默认是**NIOServerCnxnFactory**，用于创建**NIOServerCnxn**．**NIOServerCnxn**是**ServerCnxn**的一个子类，**ServerCnxn**代表客户端和服务器端之间的一个连接．

在创建完**NIOServerCnxnFactory**并配置完之后，会实例化一个**QuorumPeer**实例．**QuorumPeer**内部封装了**Leader选举**等Zab一致性算法中的内容．

![](http://upload-images.jianshu.io/upload_images/4108852-acf7da0ddf725395.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在**QuorumPeer**的**start()**方法中，主要做了这么几件事：
- 通过快照以及事务日志恢复这个节点在宕机之前的状态(**loadDataBase()**方法)
- 启动上面创建的**NIOServerCnxnFactory**
- 进行Leader选举

![](http://upload-images.jianshu.io/upload_images/4108852-adaaaf0613a5da46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，**通过快照以及事务日志恢复这个节点宕机之前的状态**，是如何做的呢？

具体是这么几步:
  - 加载SnapShot并且构建DataTree
  - 加载Transaction Log
  - 将Transaction Log应用到Data Tree
  - 提交Transaction Log并且把它们添加到**commited log**
  - 更新latest zxid

当加载SnapShot以及Transaction Log时，就涉及到了SnapShot以及Transaction Log的格式．

关于它们的格式，请参考本系列中的另外两篇文章：

[ZooKeeper源码解析(4)-TxnLog文件格式](http://www.jianshu.com/p/817602ce7128)

[ZooKeeper源码解析(5)-Snapshot文件的格式](http://www.jianshu.com/p/ee877261c54f)

这部分的代码本身也是比较简单的，这里就不深入介绍了．

由于**QuorumPeer**本身也是一个线程，我们从其**run()**方法中可以看到，它就是启动了一个死循环，不断检测这个节点的状态，然后根据不同的状态来做不同的事情，这个在后面的文章[ZooKeeper源码解析(6)-Zab实现解析](http://www.jianshu.com/p/357ca7c3b2af)中有介绍，这里就不详细说了．

需要注意的是，我们可以看到，如果当前节点处于**LOOKING**状态，那么会检测一下是否当前节点已经宕机，如果已经宕机了，那么就可以启动一个**ReadOnlyZooKeeperServer**．这个对于CAP的权衡．

如果当前节点处于**LEADING**状态，那么会在完成了确定了自己Leader的地位后，就启动一个**ZooKeeperServer**．**ZooKeeperServer**中就是一个节点，其中维护了各种数据，比如，和客户端之间连接的Session．

![](http://upload-images.jianshu.io/upload_images/4108852-0bab6834e2dc48fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你是刚开始读源码，那么可能会对**ServerCnxnFactory**,**ZooKeeperServer**,**QuorumPeer**之间的关系不清楚．

它们之间的关系如下：


![](http://upload-images.jianshu.io/upload_images/4108852-2f8cc63d49ed7cff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用的时序图如下：

![](http://upload-images.jianshu.io/upload_images/4108852-3ef733e3852556aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



其中**ServerCnxnFactory**的继承关系如下：


![](http://upload-images.jianshu.io/upload_images/4108852-93954e2bf5cac664.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ServerCnxn**的继承关系如下：


![](http://upload-images.jianshu.io/upload_images/4108852-f783bd4ed95e2edb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ZooKeeperServer**的继承关系如下：

![](http://upload-images.jianshu.io/upload_images/4108852-03a93b2e3b7f8fe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每个类图中，我都只是列出了少量属性和方法．

## 总结

总的来说，在启动时，主要是做这么几件事：

- 解析配置文件
- 根据Snapshot和transaction log在内存中构建出数据和元数据
- 创建**ServerCnxnFactory**
- 启动Leader选举过程
- 根据选举结果切换到相应的状态并进行相应的操作
- 启动一个**ZooKeeperServer**来接收客户端操作
