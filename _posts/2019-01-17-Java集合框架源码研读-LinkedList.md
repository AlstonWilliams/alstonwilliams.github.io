---
layout: post
title: Java集合框架源码研读-LinkedList
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
具体的代码，我不会在这里分析了．实际上，LinkedList的实现和ArrayList的实现基本相同，除了内部的数据结构．

LinkedList内部的数据结构是一个双向链表，它还维护了一个指向这个双向链表的指针，分别是**first**和**last**.

LinkedList提供给我们的方法，时间复杂度基本上都是O(n)．因为需要定位到具体的节点的位置．除了对LinkedList最前面和最后面的元素进行的操作．

所以说，LinkedList适合于那种只在首尾进行操作且不确定要存储的数据的大小的程序，如果在中间进行操作，时间复杂度为O(n)，而在首尾进行操作，时间复杂度为O(1)，效率还是非常高的
