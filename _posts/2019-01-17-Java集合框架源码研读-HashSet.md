---
layout: post
title: Java集合框架源码研读-HashSet
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
HashSet是一种**Set**的实现．

为什么需要Set这种数据结构呢?因为假设我们想要存储不能重复的元素，我们会怎么做?会选择哪种数据结构?数组还是链表?

选用数组或者链表，我们都需要时间复杂度为O(n)的检查操作来检查元素有没有重复．

"等一下"，你可能会说．

"我们可以选择BST啊，这样我们只需要时间复杂度为O(logn)的检查操作．"

确实可以这样做，但是还有没有更好的解决方案呢?

"当然有",　你朋友说．

他又接着说:"我们之前有学过HashMap的，为什么不用HashMap来实现呢?这样时间复杂度仅为O(1)啊．"

你反驳说:"但是HashMap是键值对的形式呀，而我们这里只是单个数据的形式，又不是键值对的形式."

"用这个数据做key不就好了嘛，value随便找个占位符放那儿不就完事了嘛!"

bingo!

就是这样，HashSet内部维护着一个**HashMap**,用这个**HashMap**存放实际的数据．然后用我们想要插入的数据做**key**,再找一个对象做占位符，当做**value**.

就是这么简单.

HashSet中,可以存放null.因为HashMap中就允许null键值对存在．

另外，由于HashSet内部的数据结构为HashMap,所以你为HashSet设置的**initial capacity**以及**load factor**同样会影响其性能．
