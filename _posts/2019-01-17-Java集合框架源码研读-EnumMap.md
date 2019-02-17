---
layout: post
title: Java集合框架源码研读-EnumMap
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
前面已经介绍了好多Map了，今天再来介绍一个，跟Enum相关的Map, EnumMap.

那么这个Map跟之前介绍的那些Map有什么区别呢?

**EnumMap**的key,必须是Enum类型的.

实际上，它实现起来非常简单.

我们可以通过**Enum**类型的**values()**方法来获取到一个**Enum**中所有数据的数组．

那这不就很简单了吗?

**EnumMap**中维护着一个key的数组(keyUniverse)和一个value的数组(vals)．由**Enum**的性质，我们可以知道，**Enum.values()**得到的数组，一定是定长的，并且其中的数据是不重复的．

所以，**keyUniverse**以及**vals**的长度都可以是固定的，没有动态扩容的问题．

所以，我们在插入数据时，比如**put(K key, V value)**方法，我们可以先获取key在原Enum中的位置，然后在**vals**的给定位置中，插入数据．


![](http://upload-images.jianshu.io/upload_images/4108852-b10e0867e5034015.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


就这样，很简单吧?

它内部也没有一些难懂的操作．

唯一有一个稍微难懂一点的方法就是**getKeyUniverse(Class<K> keyType)**:


![](http://upload-images.jianshu.io/upload_images/4108852-57ac0e081fa52f10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方法的作用就是获取**Enum**中所有的数据，并返回一个数组．

EnumMap由于其实现特性，所以，它的性能相对于**HashMap**等高一些，因为它没有碰撞的问题，也没有扩容的问题．所以基本上所有的操作的时间复杂度都是O(1).

它也允许Null值.

另外，跟其他的**Map**一样，**EnumMap**也是非线程安全的，但是它不是Fail-Fast的．

总的来说，如果你的key是一个枚举类型，那么用这个**EnumMap**要好一些．
