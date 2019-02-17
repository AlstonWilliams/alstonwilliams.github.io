---
layout: post
title: Intellij-IDEA中，调试时显示对象中字段的信息
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
今天在阅读ConcurrentHashMap的源码时，由于实在是看不懂其中的各个字段的作用，不知道到底是干什么用的，于是就想调试一下看看．

而在调试时，默认情况下，只显示ConcurrentHashMap的Map视图的表示，也就是说，默认情况下，ConcurrentHashMap是这样显示的:

![](http://upload-images.jianshu.io/upload_images/4108852-5c24b07954b07b6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这显然不能满足我们的要求啊．

经过一番折腾之后，发现可以通过下面的方法显示ConcurrentHashMap中各个字段的值:

- 首先，在你要查看的对象上，右击之后，出现一个**View as**,然后选择**Object**.

这样做完后，结果是这样的:


![](http://upload-images.jianshu.io/upload_images/4108852-3b367a7933d02705.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这基本上就能显示出来绝大多数字段了，如果还有一些字段，比如静态变量等，你想查看而又没有显示出来，那么就需要进一步设置．

- 还是在你要查看的对象上，右击之后出现**Customize Data Views**之后，在弹出的对话框中，选择那些你需要的类型:


![](http://upload-images.jianshu.io/upload_images/4108852-241276b713bc3094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后调试窗口中，显示的对象的属性就全了．
