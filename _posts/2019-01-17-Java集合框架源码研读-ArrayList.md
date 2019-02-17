---
layout: post
title: Java集合框架源码研读-ArrayList
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
在理解了ArrayList的父类**AbstractList**的实现之后，我们就要开始动手理解ArrayList.

**ArrayList**实现了**List**接口，其中定义了列表应该有的操作，还实现了**RandomAccess, Cloneable, Serializable**接口，分别让List具有能够随机读取，可复制，可以序列化的能力．

**RandomAccess**这个接口，是一个空接口，它其中没有任何方法的声明．实现这个接口，只是让我们知道**ArrayList**是可以进行随机读取的．实际上，由于**ArrayList**的内部数据结构是数组，所以它天生就具备随机读取的能力．

**ArrayList**中，有一个**DEFAULT_CAPACITY**属性，定义了**ArrayList**起始的默认长度，为10.

![](http://upload-images.jianshu.io/upload_images/4108852-d28723f5ec2e713b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但是，这并不意味着，你在创建一个**ArrayList**时，没有为其指定容量的话，它会自动为你创建一个长度为10的数组，作为内部数据容器.实际上，如果你在创建**ArrayList**时，不为其传递初始容量这个参数，其内部维护全部数据的数组，其容量还是为0.

那么**DEFAULT_CAPACITY**这个属性到底有什么用呢?

**ArrayList**还给我们提供了一个叫做**ensureCapacity**的方法，这个方法能够让我们确保手动确保数组的容量:

![](http://upload-images.jianshu.io/upload_images/4108852-289d11cdb963cf63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-8d06ce434b69b9ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结合上面的这三个方法，**DEFAULT_CAPACITY**的作用就很显然了．

在**ensureCapacity**这个方法中，如果**elementData**这个数据容器的数组，此时其中并没有任何元素，并且你传入的**minCapacity**大于10,就会将**elementData**扩容到10.反之，如果此时**elementData**中有元素，那么就会根据**minCapacity**和此刻**elementData**的长度来进行判断是否进行扩容．

在**ensureCapacityInternal**方法中，如果**elementData**此时没有任何元素，那么就将**elementData**扩容到**DEFAULT_CAPACITY**和**minCapacity**中较大的那个．

需要注意的是，数组的长度，和数组中元素的个数并不是一回事．数组的长度，是在数组刚开始被创建时就确定下来的，这样JVM才能为其分配空间并初始化．而数组中元素的个数，则如其名字所示．

比方说，我们这样创建一个数组:

**int[] arr = new int[10];int[0] = 0;int[1] = 1;**

此时，数组的长度为10，但是其中元素的个数仅为2.

那么，一个**ArrayList**的最大长度是多少呢?从源文件中，我们可以看到为**Integer.MAX_VALUE - 8**．那么为什么要减8呢?

从StackOverflow上面得知，由于数组的索引采用int表示，所以理论上说，数组的最大值应该是**Integer.MAX_VALUE**或者**Integer.MAX_VALUE - 1**或者**Integer.MAX_VALUE - 2**．具体多大取决于JVM.

而可能会有一些JVM会将某些ArrayList的信息写入到其内部的数组中，所以，为了安全起见，就让其最大容量为**Integer.MAX_VALUE - 8**.这只是我的个人猜测，不保证正确．我们从源文件中看到的解释为:


![](http://upload-images.jianshu.io/upload_images/4108852-f112a69b4fbe8db5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**ArrayList**还有两个类似的属性，一个为**EMPTY_ELEMENTDATA**，另一个为**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**.我们从源文件看到对它们的解释如下:


![](http://upload-images.jianshu.io/upload_images/4108852-916c8f4b1e7065ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那么这两个属性有什么区别呢?

其实它们有啥区别我也不清楚，反正两个都是一个空的数组，而且这两个空的数组在使用的过程中也都没有改变，有啥不一样的!

**ArrayList**有三种初始化方式，一种是传入初始长度，然后ArrayList将其内部的**elementData**初始化为指定长度的数组，另一种是什么参数也不传，这样的话，**ArrayList**就会将其内部的**elementData**初始化为一个空数组，最后一种初始化方式，是传入一个**Collection**对象，**ArrayList**会先将这个**Collection**对象转换为一个数组，然后将其赋给**elementData**.

我们再来看一下**ArrayList**的扩容函数，**grow**.


![](http://upload-images.jianshu.io/upload_images/4108852-4d491cc8a812d161.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-24d87d7601f5745d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从这个函数中，我们可以看到，**ArrayList**每次进行扩容时，都会先计算我们想要的最小容量是否小于原数组的长度的3/2倍，如果不是，则扩容到**minCapacity**大小．如果是，则扩容到**原数组的长度的3/2倍**．所以说，我们在使用**ensureCapacity**这个函数确保数组的长度时，数组扩容后的长度并不一定是我们指定的**minCapacity**,而更可能是**原数组长度的3/2**倍．

另外，这个方法中，很细节的地方是，在计算3/2倍时，它采用的是**oldCapacity + (oldCapacity >> 1)**的方式，各位都知道这是什么意思．对于3/2这么一个不快的增长速度来说，如果需要频繁进行扩容，这就是一个不错的优化点．如果每次增大为原来的两倍，那么，在第10次扩容后，**elementData**的长度将为**10240**.而每次仅扩容为原来的**3/2**倍，第十次扩容后，**elementData**的长度将为**570**.很显然，远比每次扩大为两倍小得多．

我想这里，是开发者为了协调空间浪费和时间而做的折中．如果我们开发应用时，**ArrayList**需要很快的进行扩容，这里我们可以调整一下扩容的速率．

我们上面提到过，数组的长度，只能在使用前指定，然后JVM才能进行内存分配以及初始化，也就是说，在初始化完数组之后，实际上，它的长度就是不可变的了．而每次扩容，就会创建一个新数组，对于急速增长的**ArrayList**，效率上是否有点问题,如果我们指定的**minCapacity**一直都小于**oldCapacity * (3 / 2)**的话?

接下来，我们看一下**indexOf**方法的实现:


![](http://upload-images.jianshu.io/upload_images/4108852-2422f874ca38a32e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到，**indexOf**方法的实现，跟**AbstractList**中的实现一样，时间复杂度都是**O(n)**.所以，如果我们做应用时，用**ArrayList**来存储数据，并且**ArrayList**还不小，还有热点数据的需求．那么最好写一个Map来记录热点数据和其在**ArrayList**中对应的位置关系．这样，如果我们需要查询该热点数据m次，则使用平摊分析可得时间复杂度为O(n/m)．

但是，这样也有一个问题，就是，我们知道，ArrayList中一旦删除了数据之后，那么在**elementData**中，位于该特定数据之后的数据，都会向左移动．这样就会有映射失效的问题．

所以，这不是一个好的缓存方案．

我想各位此时心里一万个草泥马飘过...

我也是边写边分析的...

起码现在我们知道了为什么这样做缓存不好了，对吧?

其实我们还是可以这样做的，但是在删除**ArrayList**中的数据的时候，我们需要同时检查Map中，是否有需要移动的元素，然后Map中存储的它们在ArrayList中的索引 - 需要移动的值．这些还需要放在同一个事务中来做．

你现在心里可能会想，这不是脱了裤子放屁嘛．我直接用一个HashMap来缓存不就好了，干嘛还要整个ArrayList.

好像确实是这样...

所以，上面跟缓存相关的这部分，似乎都是一本正经的扯淡...

回归正题!

另外，每次删除一个元素，都需要移动此元素后面的元素，这样效率是否有问题呢,对于那种频繁删除ArrayList中的元素的应用来说?


![](http://upload-images.jianshu.io/upload_images/4108852-30c28af975b473bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我并不知道它具体在移动时是采用什么算法的，**System**的**arraycopy**这个方法是native的．

另外，**ArrayList**的Iterator是Fail-Fast的．也就是说，只要我们调用了那种会改变数组的长度的方法，那么**Iterator**就不会正常工作了，而会抛出异常了．

还有**ArrayList**的**subList**方法，需要注意的是，它返回的并不是一个新的List,而是一个原来的**ArrayList**的包装类．所以，如果你对这个返回的**SubList**进行修改的话，实际上就是在修改原来的**ArrayList**.

其他的方法就基本上跟**AbstractList**差不多了．

**ArrayList**还提供了一个叫做**ArrayListSpliterator**的类，这个类用于对**ArrayList**中的元素，来实现并行计算．具体的实现我也没有深入了解过．

总体来说，ArrayList的查找,修改，增加的效率还是蛮高的，特别是对于查找和修改，是O(1)的时间复杂度，增加的话，对于数据增长不是特别迅速的场景，也是O(1)，但是一旦要进行扩容的话，就是O(n)了，删除元素的话，就是O(n)了．

所以，如果是那种读多写少，并且确定数据的数量不会超过**Integer.MAX_VALUE - 8**的场景的话，ArrayList还是蛮不错的．
