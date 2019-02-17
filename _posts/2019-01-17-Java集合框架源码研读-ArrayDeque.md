---
layout: post
title: Java集合框架源码研读-ArrayDeque
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
前面介绍过一个队列的实现-**PriorityQueue**，现在我们介绍一下**ArrayDeque**.

从它的名字中，我们可以看到，其内部结构是一个数组，并且它是一个Deque.

我们首先看一下这个类的文档注释:

![](http://upload-images.jianshu.io/upload_images/4108852-17e9da79a9a5fbb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从中我们可以提取出来几点重要信息:

- 内部数组的容量会自动增加，如果必要的话
- 这个类不是线程安全的
- 这个类是Fail-Fast的
- 如果把它当做栈来使用，那么它比Stack这个数据结构更快(这点我也不知道是什么原因，我感觉它应该比Stack慢．因为Stack只需要在栈顶增加和删除数据，时间复杂度均为O(1),而ArrayDeque在删除数据时由于还要涉及到数据移动的问题，所以即使单纯的增加和删除数据的时间复杂度为O(1)，但是再加上数据移动的开销，其不是应该比Stack慢吗?)
- 如果把它当做队列来使用，那么它比LinkedList更快(这点也不理解，因为LinkedList中有一个指向链表末尾的指针，并且每个节点都有指向前一个节点的指针，那么在插入数据时时间复杂度也为O(1)啊)
- 它的所有操作的时间复杂度基本上都是O(1),除了一些需要通过遍历找到数据位置的操作，比如**remove(Object), removeFirstOccurence(Object), removeLastOccurrence(Object), contains(Object)**等．

## ArrayDeque的内部数据结构

其内部数据容器为一个数组, **Object[] elements**.

然后还有两个属性，**head**和**tail**，分别表示**elements**中，第一个数据所在的位置以及最后一个数据所在的位置．

## ArrayDeque的重要操作

ArrayDeque中，有这么两个方法，用于扩容:


![](http://upload-images.jianshu.io/upload_images/4108852-eb83a3f29ca5d43c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这一个方法用于将容量扩展两倍并拷贝数据.

![](http://upload-images.jianshu.io/upload_images/4108852-fbdf13b32dbcf010.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方法用于找到跟**numElements**接近但是比它大的2的整数次幂．如果**numElements**已经是2的整数次幂，则取它的两倍.

其他的方法就很简单了．就是向队首和队尾插入删除数据的一些方法．这里就不一一介绍了．
