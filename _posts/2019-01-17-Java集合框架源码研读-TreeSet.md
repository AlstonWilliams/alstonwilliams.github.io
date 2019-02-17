---
layout: post
title: Java集合框架源码研读-TreeSet
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
其实Java集合框架中的很多类的设计思想，都是相同的．

比如，前面介绍Map时，我们介绍了**HashMap, LinkedHashMap, TreeMap**,现在介绍Set,我们前面也介绍过了**HashSet, LinkedHashSet**.现在又来介绍TreeSet.

那么为什么我们需要TreeSet呢？

如果你认真读过我前面的关于Java集合框架的文章，那么这里想必你很容易回答上来．

因为尽管**LinkedHashSet**能够让我们按照元素插入时的顺序来进行遍历，但是有的场景下，我们可能需要让插入**Set**的元素有序，出于这个目的，提出了**TreeSet**.

那么这个类是如何实现的呢?

**HashSet**的数据容器是**HashMap**, **LinkedHashSet**的数据容器是**LinkedHashMap**,那你说，**TreeSet**内部的数据容器是什么?当然是**TreeMap**.

由于**TreeMap**的性质，其不能接受null值．

跟**TreeMap**一样，它还提供了**lower(E e), floor(E e), ceiling(E e), higher(E e)**方法，分别用于获取**小于特定key的那个key,小于等于特定key的那个key,大于等于特定key的那个key,大于特定key的那个key**.

Ok.就这么简单．
