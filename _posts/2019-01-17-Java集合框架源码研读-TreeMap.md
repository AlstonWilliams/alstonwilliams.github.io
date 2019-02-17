---
layout: post
title: Java集合框架源码研读-TreeMap
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
前面我们已经介绍了两个**AbstractMap**的实现了，分别是**HashMap**和**LinkedHashMap**.我们也看到了，**LinkedHashMap**是**HashMap**的一个优化版本，它能够根据元素的插入顺序或者元素的访问顺序来进行遍历．

那么今天要介绍的**TreeMap**又是什么鬼?

## 简介

![](http://upload-images.jianshu.io/upload_images/4108852-d8a562c3af5b71ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们这里直接贴出文档首部的说明．从说明中我们可以看到，我们关注的主要有三点：

- 这个map能够根据key进行排序
- 它是基于Red-Black tree实现的
- **containsKey(),  get(),  put(),  remove()**等方法的时间复杂度为O(logn)

就是这样的一个map而已.

注意上面的第三点，我们知道，**HashMap**和**LinkedHashMap**中，这些操作的时间复杂度基本都为O(1)．所以，如果不是要求按Key的顺序进行排序的话，是没有理由使用这个map的．本文后面还会给你不使用这个map的理由．

这里还有一点要说明，就是为什么采用Red-Black tree,而不是BST或者AVL.关于这一点，我们需要理解Red-Black tree相对于后两者的优势．请参考[Java集合框架源码研读-HashMap](http://www.jianshu.com/p/e03f1bd170e7)中的相关说明来理解．

## 结构

**HashMap**，有一个数组用于存放键值对，如果发生碰撞，则用链表或者Red-Black tree来存放碰撞的元素，采用链地址法来解决冲突问题．

而在**TreeMap**中，则仅有一棵Red-Black Tree而已，我们看一下其中的每个Node的定义:


![](http://upload-images.jianshu.io/upload_images/4108852-0871683a538b050d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从其定义中，我们可以看到，在碰撞时，它并没有一个链表来存放碰撞的元素．那么它是如何解决碰撞的呢?下文中将会给你答案.

## 关键操作

我们来看三个关键操作的实现，分别是**get(),  put()**方法和用于按key的顺序访问的实现．

#### get()方法的实现

**get()**方法实际上是调用的**getEntry(Object key)**,所以这里我们仅仅贴出这个方法的实现．


![](http://upload-images.jianshu.io/upload_images/4108852-19f38324cbbca008.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个应该没有什么好说的吧?仅仅是一个中序遍历而已．学过数据结构中的**树**的朋友应该都很容易理解．

#### put()方法的实现


![](http://upload-images.jianshu.io/upload_images/4108852-08441939d1fb43d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-50039ae35487ff5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很简单，就是中序遍历找到特定节点，然后替换其value.

这里就给出了前面如何解决冲突的答案．出现冲突时，并不会采用**链地址法**或者**开放定址法**等方法来解决冲突，而是直接替换掉对应节点的value.这就很尴尬了，万一在你的应用场景中，出现了冲突的情况，在你不知情的情况下，就会替换掉对应的值．

其实这个问题也不是什么大问题．因为比较器**comparator**应该是你自己定义的，也就是说，你应该意识到其实它是不能存放重复的元素的．

另一个需要注意的地方就是，**不允许key为null**．

#### 有序遍历是如何实现的?

查看keySet()方法，一直向后寻找，我们能够发现其中最重要的方法是**successor(Entry<K,V> t)**.

![](http://upload-images.jianshu.io/upload_images/4108852-4248a37c4df9b207.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很容易的就能发现，这实际上就是一个中序遍历．

但是，中序遍历得到的结果不应该是有序的呀，前序遍历得到的结果不应该才是有序的吗?

为什么呢?

因为实际上第一次调用**successor(Entry<K, V> t)**时，实际上传入的就是这棵Red-Black Tree的最左节点.

![](http://upload-images.jianshu.io/upload_images/4108852-855a826b94021d3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-15315cd736515b77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看图中**keyIterator()以及getFirstEntry()**的实现．

理解了吧?

## 后记

其实TreeMap中还提供了很多其他的方法，比如获取跟传入的key最接近的键值对等．

核心方法上面已经介绍了，其他的方法，各位自行探索吧.
