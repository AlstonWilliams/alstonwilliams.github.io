---
layout: post
title: HDFS源码解析(4)-DataNode删除过期block
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在上篇文章中，我们最后抛出一个问题，即DataNode怎么知道那些block是corrupt的，或者是过期的，并删除它。

在这篇文章中，会介绍这个过程，其实很简单:

![](https://upload-images.jianshu.io/upload_images/4108852-1817096f10ede096.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就是DataNode进行block report，然后NameNode检查是否有corrupt或者invalidate的block，如果有，就给DataNode发送一个`BlockCommand.INVALIDATE` Command来删除它。

就是这么简单。

有一点需要注意的是，默认情况下，DataNode每过一个小时才会进行一次Block report。所以，如果这个DataNode在这一个小时中，过期的block占满了磁盘，那么这个DataNode就不能提供存储服务了。

不过这种极端情况应该极少极少。
