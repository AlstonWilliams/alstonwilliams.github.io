---
layout: post
title: Java-ArrayBlockingQueue(译)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
**ArrayBlockingQueue**是**BlockingQueue**接口的一个实现.

**ArrayBlockingQueue**是一个有大小的**BlockingQueue**实现.也就是说,你不可以在里面存储无限多的元素.在实例化它的时候,你指定了它的大小,之后就不可以更改了.

**ArrayBlockingQueue**内部使用**FIFO**算法来存储元素.

我们可以使用下面的代码,来实例化一个**ArrayBlockingQueue**并使用它:

![](http://upload-images.jianshu.io/upload_images/4108852-c2f32ba017f22bcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
