---
layout: post
title: Java集合框架源码研读-HashMap
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
> 写这篇文章之前，当我在研究HashMap的源码的时候，发现源码中有几处用到了一些我完全理解不了的算法，于是，从网上搜索相关的内容，期望能够理解这些算法，以及为什么这样做是正确的．
最后，误打误撞，发现了一篇跟HashMap工作原理相关的文章．找到这篇文章之后，我也是参考着这篇文章来深入的理解了一遍源码．这篇文章非常浅显易懂，如果你有尝试过阅读HashMap的源码的话．
有了那篇文章，本不想写这篇文章的，因为那篇文章其实已经解释的很透彻了．但是，在自己阅读完源码之后，还是有一些其他的收获的．所以，就有了此文，当做对那篇文章的补充．

这里首先贴出那篇文章的地址:[Java HashMap工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

不过似乎github.io在国内被墙了，如果各位不翻墙的话，这篇文章可能是看不了的．

先贴出一张总结之后产生的图:

![](http://upload-images.jianshu.io/upload_images/4108852-ab7d348eb91894b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于英语水平有限，其中有些地方表达的可能不太通顺．但是意思应该是差不了．后面也介绍了这些内容，所以各位可以略过此图不看．

我们先介绍一下HashMap的总体结构，然后我们在对这个结构以及一些重要的操作进行详细的介绍．

## HashMap的总体结构

HashMap内部，其实由一个数组，以及一条条链表组成．

我们在学习数据结构和算法时，在Hash这一部分里，针对冲突碰撞，最简单的办法应该就是，将冲突的哈希对组成一张链表．也就是上图中的上面的那一部分．

这个相对还是比较容易理解的．还有其他的处理碰撞的措施，比如再Hash，直到找到一个空闲的位置放进去，这种方法，我现在还是不能理解那我们进行查找时，应该怎么办．你怎么直到要插入时是直接插入进去的，还是经过一次碰撞做了一次再hash插入进去的，还是经过两次碰撞做了两次再hash插入进去的?

当然，可以再有一个数据结构来记录hash次数的信息，但是这样不是多此一举吗?

我们再来看一下Node的组成，Node就表示一个键值对.

- hash. Key的hash值
- key
- value
- next.指向碰撞的下一个键值对．

很容易理解吧?

基本结构就是这样．当然，也有变体．就是如果一条链表中的Node太多了，也就是有太多Node碰撞了，由于是链表，所以查找性能是O(m)，其中m是此链表的长度，当链表过长时，查找性能不行，所以，优化了一下，当此链表中的长度太长时，就将此链表转换为一棵Red-Black Tree.这样其查找性能就是O(logm)了．

## HashMap内部的属性

好，大体结构我们介绍完了，现在我们就来介绍一下其内部的各种属性的作用.

#### 常量部分:

- **DEFAULT_INITIAL_CAPACITY**:代表默认情况下，**table**的长度.默认情况下是16.
- **MAXIMUM_CAPACITY**: 代表**table**的最大长度.默认情况下是2^30.
- **DEFAULT_LOAD_FACTOR**: 代表默认的负载因子．那么什么是负载因子呢?它是衡量什么时候**table**该扩容的一个指标．为什么需要这样单独的一个属性呢?我觉得是处于这样一种目的:当**table**中的元素越来越多时，碰撞的几率也就越来越高，所以为了保证不会太频繁的碰撞，就用这个属性来保证当**table**中的Node到了一定数量时，就进行扩容．默认情况下是0.75f.
- **TREEIFY_THRESHOLD**: 代表树化的阈值，也就是链表的长度大于多少时进行树化，默认情况下为8
- **UNTREEIFY_THRESHOLD**:什么时候进行取消树化．默认情况为当树中只有6个元素时.
- **MIN_TREEIFY_CAPACITY**:进行树化时,**table**的最小长度．应当至少是**4 * TREEIFY_THRESHOLD**，以避免**table**扩容和树化之间的冲突.默认情况下是64.
 
#### 全局变量部分

- **table**:　这个就是我们上面说的那个数组，就是用来保存没有碰撞的Node,也就是链表中的第一个Node的．
- **size**: Map中保存的键值对的总数，注意不是**table**中保存的Node的总数.
- **modCount**:Map结构修改的次数，用于让Iterator Fail-Stop.
- **threshold**: **table**进行扩容时的阈值,默认情况下为**table.length * loadFactor**
- **loadFactor**:　实际的负载因子.

## HashMap内部重要的函数

#### static final int hash(Object key)

![](http://upload-images.jianshu.io/upload_images/4108852-71ac074b59323a55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个函数用于计算key的hash值．

从源码中，我们可以看到计算方式为，把key的hashCode()得到的哈希值，先把h的高十六位移到第十六位上，然后让这两个值做异或运算．

#### static final int tableSizeFor(int cap)

![](http://upload-images.jianshu.io/upload_images/4108852-f9c8e3c99b4a2497.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用于计算最接近cap的并且比它大的二的整数次幂的数．

关于这个算法的解释，请移步[这里](http://blog.csdn.net/fan2012huan/article/details/51097331).

其实自己多找两个数，就照着这个算法走一遍，就能清楚为什么了．

#### final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)


![](http://upload-images.jianshu.io/upload_images/4108852-08db3d81dbc8d3fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-9d026c7c7feaf07f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-ffa3913d1910c277.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方法，可以说是整个HashMap的核心了．我们通常调用的**put(Object key, Object value)**方法，其实都是调用的这个方法．

从这个方法中，我们可以看到，如果在插入键值对时，**table**为空，则对其进行扩容．然后，通过**(n - 1) & hash**来计算这个键值对在**table**中的索引．如果此时该索引处没有Node,也就是没有碰撞，则初始化一个Node.如果进行碰撞了，则判断我们要插入的键值对的key,是不是跟碰撞的那个相同，如果相同则修改此键值对的value.如果发生碰撞，并且此链表已经被树化过，那么就插入一个TreeNode,如果没有进行过树化，则查看是否到达树化的阈值，如果到达就进行树化．

那么为什么要根据**(table.length - 1) & hash**来计算这个键值对在**table**中的索引呢?因为如果不这样计算，那么可能由于**hash**太大，比**table.length - 1**还大，HashMap就不知道要将这个键值对放到哪里．而通过上面那种方式则可以确定．

这样，我们也很容易发现一个问题，就是如果**table.length - 1**太小，则很容易发生碰撞．作者解释说，由于现在大多数对象的hashCode()已经分布的很均匀，并且可以通过树化来解决这个问题．所以这个问题不是问题．

#### final Node<K, V>[] resize()


![](http://upload-images.jianshu.io/upload_images/4108852-fd94f3c2c7f8569a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-a455908da4e5ff06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-acf2dbf63a8e12ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-1030f91e95387318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方法其实也是HashMap中的一个关键的方法，它的实现也挺容易理解的．简单来说，就是将原来的**table**的容量扩大为两倍，然后相应的扩大**threshold**，再创建一个新的容量为原来的**table**的容量的两倍的新数组，然后将原先存在于**table**中的键值对，重新进行哈希运算，并移动到新的数组中．

## HashMap的一些其他思考

#### 为什么在树化时，是初始化为Red-Black Tree，而不是BST?

因为在极端情况下，BST的时间复杂度为O(n).而Red-Black Tree不管在什么条件下，都保证时间复杂度为O(logn)．性能相对于BST好．

### 那为什么不树化为AVL呢?

我们知道BST的时间复杂度为O(n)是在BST是一棵完全的斜树时发生的．那么AVL由于其保证平衡性，自然没有这种顾虑．那为什么不树化为AVL呢?

这是由于，AVL为了要保证插入数据或删除数据时，各个左右子树的高度差不能超过1的这个特性，所以就可能要进行很多左旋，右旋操作．而这个操作是很复杂的．所以AVL主要是用于读多写少的场景．

而Red-Black Tree则不相同了．由于其相对较少的旋转操作，相对与AVL来说，插入数据时，性能更高一些．

#### 为什么下面的代码得出的hashCode()是相同的?

**HashMap<String, String> hashMap1 = new HashMap<String, String>();
HashMap<String, String> hashMap2 = new HashMap<String, String>();
hashMap1.put("1", "2");
hashMap1.put("2", "1");
hashMap2.put("2", "1")
hashMap2.put("1", "2");**

这是由于HashMap的无序性．

由于HashMap的hashCode()方法并没有自己实现，而是继承父类**AbstractMap**的hashCode()方法，结合**AbstractMap**的hashCode()方法，我们发现，即使你添加的顺序不同，其得到的hashcode，总是相同的．

这也是应该的，因为HashMap并不保证以我们插入数据的顺序来保存数据，也就是说，HashMap是无序的．

所以，既然它是无序的，我们在比较两个HashMap时，在计算hashCode时，自然就无需考虑其中元素的保存的顺序．

#### 为何下面的HashMap的hashCode()总是0?

**HashMap<String, String> hashMap1 = new HashMap<String, String>();
hashMap1.put("1", "1");
hashMap1.put("2", "2");**

当key和value相同时，对应的键值对的hashcode就是0.


![](http://upload-images.jianshu.io/upload_images/4108852-1de32b00b6f0aa38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## HashMap的其他需要注意的点

- HashMap不是线程安全的
- HashMap是无序的．它不会根据你插入的哈希对的顺序来存储键值对
- HashMap允许null键值对
- HashMap在进行扩容时，其中的已经存在的哈希对会进行重新哈希，重新定位
