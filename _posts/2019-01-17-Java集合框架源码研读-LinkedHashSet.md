---
layout: post
title: Java集合框架源码研读-LinkedHashSet
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
在上一篇文章中，我们介绍了HashSet.今天我们就来介绍一下**LinkedHashSet**.

其实**HashSet**和**LinkedHashSet**的关系，就跟**HashMap**和**LinkedHashMap**的关系一样．

**LinkedHashSet**是通过实例化一个**LinkedHashMap**来实现按序访问，只不过**LinkedHashSet**不允许我们指定按照哪种顺序进行排序，而只是默认按照元素插入的顺序排序．

关于**LinkedHashMap**的实现原理，请参考我的文章:[Java集合框架源码研读-LinkedHashMap](http://www.jianshu.com/p/3ef79f42ef52)．

我们看**LinkedHashMap**的源码，从中根本就看不到跟存储元素插入顺序相关的任何数据结构．

其实，其实现的重点在于其构造函数上，我们看一下其构造函数:


![](http://upload-images.jianshu.io/upload_images/4108852-51e5e5a1c43edfe9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


由于**LinkedHashSet**的父类是**HashSet**,所以我们查看一下**HashSet**中相关的构造函数：


![](http://upload-images.jianshu.io/upload_images/4108852-80fa2a38bfd89aa1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到了吧?就是初始化的一个**LinkedHashMap**．
