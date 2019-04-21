---
layout: post
title: 当Java的枚举类型遇到Spark
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
其实不仅仅是Spark，也适用于其他的分布式技术。

假设我们有这么一个Java枚举类:
~~~
public enum AtomicType {
  NORMAL, CHECK
}
~~~

在本机上测试一下它们的hashcode，你会看到它们是相同的。

但是，要注意，这并不意味着，在分布式环境中，它们也是相同的。

我们先来说明枚举类的hashcode是怎样生成的。

我们进入到`Enum`这个类，看它的`hashCode()`方法，可以看到，最终它是调用了`Object.hashCode()`。而`Object.hashCode()`，我们从它的注释中，可以看到，是由内存地址转换来的。

![](https://upload-images.jianshu.io/upload_images/4108852-d29022b0f3c7e86a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，当我们在Spark的Executor进行操作后，返回一个涉及到枚举类型的key，然后执行aggregateByKey()，就会发现，看起来明明是一模一样的key，为什么会被当成两个。

其实不仅仅是Spark，其他的分布式技术，也要注意这个问题。小小的一个bug，害我调了两天，夜不能寐，好不容易才调好。
