---
layout: post
title: Java集合框架源码研读-LinkedHashMap
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
上一篇文章中，我们已经介绍了一个很重要的集合类-HashMap.我们也提到了它的一个特性，就是不是按我们插入的顺序来读取元素．那么，今天我们就来介绍一个能够按照我们插入的顺序来读取元素的HashMap的变体，LinkedHashMap．

其实LinkedHashMap和HashMap的实现并没有什么差别，LinkedHashMap是HashMap的一个子类．从源码中，我们可以看到，很多方法，比如**put()**方法，在LinkedHashMap中是看不到的．

今天我们也不具体谈论LinkedHashMap的实现细节，只是讨论一下它是如何实现我们能按照插入元素的顺序或者读取元素的顺序的顺序进行遍历．

为了实现这一点，LinkedHashMap修改了**HashMap.Node<K,V>**的实现，如下所示:

![](http://upload-images.jianshu.io/upload_images/4108852-ad614448e060a786.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，它将**HashMap.Node<K,V>**变成了一个双向链表中的一个节点．

并且，**LinkedHashMap**还专门维护了这么几个属性:

![](http://upload-images.jianshu.io/upload_images/4108852-acdc202d24e87468.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其中，**head**指向这个双向链表的首节点，**tail**指向这个双向链表的尾节点．**access**表明这个双向链表应该是以什么顺序来保存元素，如果这个属性为true,则按访问顺序来保存，如果为false，则按元素的插入顺序来保存．

什么意思呢?

有下面代码:

**LinkedHashMap<String, String> linkedHashMap = new LinkedHashMap<String, String>();
linkedHashMap.put("1", "1");
linkedHashMap.put("2", "2");
linkedHashMap.put("3", "3");
linkedHashMap.get("2");
linkedHashMap.get("3");
linkedHashMap.get("1");**

如果**access**为true,那么此双向链表为**2,3,1**．如果**access**为false,则此双向链表为**1,2,3**.

嗯．有了这个数据结构之后，LinkedHashMap又是如何做的呢?

首先，它重写了**HashMap**的**get(Object key)**方法，其实现如下:


![](http://upload-images.jianshu.io/upload_images/4108852-5290907332ba66bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，跟**HashMap**的**get(Object key)**相比，这里多了几行:

**if(access) afterNodeAccess(e);**

那**afterNodeAccess(e);**又是什么鬼?


![](http://upload-images.jianshu.io/upload_images/4108852-60c7502afd992a32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**afterNodeAccess(Node<K,V> e)**是**LinkedHashMap**中新增的一个操作，它的作用是，当访问元素时，将元素添加到双向链表中．

除此之外，**LinkedHashMap**中还添加了另外两个方法:**afterNodeRemoval(Node<K, V> e)和afterNodeInsertion(boolean evict)**方法，分别用于在元素被删除以及元素被插入后，进行处理．其定义如下:

![](http://upload-images.jianshu.io/upload_images/4108852-cfdbe6883ecdd154.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看看上面的实现，很简单，对吧?

接下来我们看看如何按照元素插入的顺序排序.

上面我们也说过，**LinkedHashMap**中，其实并没有重写**HashMap**的**put(Object key, Object value)**方法．那么它是如何做到的呢?

要了解这一点，我们首先需要了解**HashMap**中**putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)**方法的实现，因为**put(Object key, Object value)**实际上就是调用的这个方法．

![](http://upload-images.jianshu.io/upload_images/4108852-6e9aab5319311e9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意上面重点标注的那一行．

从实现中我们可以看到，如果插入时，**table(table是HashMap中的一个属性，用于存放哈希对，详情请参考上篇文章)**中没有发生冲突，也就是这个key之前是不存在的，则**newNode(...)**,如果发生冲突，也要**newNode(...)**.

那么，**LinkedHashMap**只要重写**newNode(...)**这个方法，就能实现按照元素插入的顺序来保存到双向链表中了．

实际上，**LinkedHashMap**也确实是这么做的．


![](http://upload-images.jianshu.io/upload_images/4108852-ac74289fbfa2e9e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**linkNodeLast(p)**的意思，顾名思义，就是将新创建的节点插入到双向链表的末尾中去.


![](http://upload-images.jianshu.io/upload_images/4108852-a85c4f3ddd0d380e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这就理解了**LinkedHashMap**是如何实现检索时有序的了吧?

需要注意的是，**LinkedHashMap**中的这个双向链表，只是用于确定顺序，节点在插入时，要插入到**table**中的哪个位置，跟它可没有关系，还是得通过**table.length & hash**来计算．

既然检索时可以有序，但是我们也没有看见**LinkedHashMap**为我们提供一个**get(int index)**的方法，来进行有序的检索啊．那有序检索是如何体现的呢?

答案就是，通过迭代器来实现有序检索．


![](http://upload-images.jianshu.io/upload_images/4108852-532efd65321125f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一目了然吧?

从迭代器的**nextNode()**方法中，我们可以看到，当**modCount != expectedModCount**时，会抛出**ConcurrentModificationException**.那么什么情况下**modCount != expectedModCount**以及为什么不等于呢?

当我们按照访问的顺序排序，并且在用迭代器有序访问元素时，此时如果我们调用**get(Object key)**，就会造成**modCount != expectedModCount**.从上面的**afterNodeAccess(Node<K, V> e)**，我们可以看到末尾有一句**modCount++**.

当我们按照插入的顺序排序，并且在用迭代器有序访问元素时，如果此时调用**put(Object key, Object value)**,也会造成**modCount != expectedModCount**.从**HashMap**的**put(Object key, Object value)**方法的实现中我们就能看到．

那么为什么要这样做呢?

因为如果我们在迭代过程中，双向链表中增加了或者删除了元素，导致它的结构发生了变化，那么我们在迭代过程中可能就会遇到一些莫名其妙的错误，所以，为了防止这种情况，就先在**nextNode()**方法中，先查看一下双向链表的结构有没有发生变化．

就是这个样子....
