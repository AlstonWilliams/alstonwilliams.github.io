---
layout: post
title: Java集合框架源码研读-AbstractList
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
本想先阅读ArrayList的源码，但是ArrayList继承于AbstractList这个抽象类，于是就先阅读这个抽象类了．

在这个抽象类中，我们可以发现，很多方法的实现，都是抛出一个**UnsupportedOperationException**异常，等待具体的实现类来实现．

![](http://upload-images.jianshu.io/upload_images/4108852-51095eb0331600d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在**indexOf**方法的实现中，我们可以看到，它是采用的顺序遍历的方式，这是最常见但是同时效率也是最低的遍历方式．



![](http://upload-images.jianshu.io/upload_images/4108852-efd9e0b5638c1ac9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



同时，我们也能看到，AbstractList是允许其中的元素为null的．如果找到元素，则返回其所在的位置，否则返回-1.

在各种查找算法中，性能最好的，似乎就是O(logn)了．但是这些算法都需要数据是有序的．那么存不存在一种算法，即使数据是无序的，其时间复杂度也比O(n)好呢?

确实存在，那就是哈希算法，如果能够确定有效的哈希函数，那么查找性能是O(1)．远比O(n)要好的多．可是，如何设计这么一个高效的哈希函数呢?这一部分，我想会在以后研究**HashMap**的时候，解答疑惑．

但是，使用这种方式，我们能保证，读取的时候，如果按照索引来读取，读取到的元素的顺序，跟元素被插入时的顺序相同．而如果使用哈希函数，则不能保证这一点．

AbstractList中，还为我们提供了一个获取元素的最后出现位置的方法，跟上面那个获取第一次出现位置的方法没什么不同，只不过是从后面开始遍历的．

除此之外, AbstractList还提供了一个iterator()方法，这个方法相信各位已经很熟悉了，它会获取一个实现了Iterator接口的Itr类用于迭代当前的List.

但是，它不是线程安全的．也就是说，如果我们已经获取到了Iterator了，而此后这个List被其他的线程修改了，那么会抛出运行时异常．

它还提供了一个listIterator()的方法，它会返回一个实现了ListIterator接口的ListItr类.那么Itr和ListItr有什么区别呢?

我们从名字上应该也能看出来，Itr类是ListItr的父类，它实现了**Iterator**接口，并实现了了一下几个方法:**hasNext, next, remove, checkForComodification**.我们可以看到，它只能向后遍历(这里称向索引大的元素遍历称为向后遍历),并且，只能从数据容器的起点开始读，也只能移除某个元素，而无法在迭代的过程中，重新设置此元素的值．

而ListItr呢，继承了Itr的那些特性，同时通过实现**ListIterator**类，增加了自己的特性．它相对于Itr增加了如下方法:**hasPrevious, previous, set, add**.从这些函数的名字中，我们就可以知道，它支持向前遍历，在遍历的过程中重新设置元素的值，以及在遍历过程中增加元素．

并且，我们可以看到它的构造函数声明如下:


![](http://upload-images.jianshu.io/upload_images/4108852-6bc3d6cead2970b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到，我们可以从指定的位置开始进行迭代．

在Itr和ListItr的实现中，我们可以看到，基本上每个方法的实现，都会调用一个叫做**checkForComodification()**的方法．这个方法是干什么的呢?Itr和ListItr中，都会维护一个变量，叫做**expectedModCount**，它记录了它认为List被修改的次数，刚开始时，它被初始化为List被修改的次数．**checkForComodification()**方法，通过将这个变量，和另一个表示List实际被修改的次数的叫做**modCount**的变量比较，就能得知，在获取到Iterator之后，List是否被修改过，进而抛出**ConcurrentModificationException**异常．

当然，前面也介绍了,**Itr和ListItr**实际上也可以在遍历过程中修改**List**，所以我们在那些修改**List**的结构的方法中，就需要再同步一下**expectedModCount**和**modCount**的值．防止是由于**Iterator**本身修改了List而导致抛出**ConcurrentModificationException.**

以**ListItr**的**remove**为例:

![](http://upload-images.jianshu.io/upload_images/4108852-6ae2841d7d5a7b07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那**modCount**这个变量的值都是在什么情况下会被修改呢?

当我们的子类在实现**AbstractList**时，对于那些修改List结构的方法，比如会造成List的大小发生了变化的函数，那么我们可以在这些方法内部，让**modCount**+1.来实现让**Iterator**发现List被修改过并抛出错误的功能．

我们为什么需要让**Iterator**发现**List**被修改过并抛出异常呢?这是为了防止在并发的环境下，造成不确定的问题．

**AbstractList**还给我们提供了一个**removeRange**方法，通过这个方法，我们可以删除一定范围内的元素，并将此范围右侧的元素左移(但是我并没有发现它实现了将元素左移的功能).此方法接受两个参数，一个是**fromIndex**,另一个是**toIndex**，分别代表要删除的元素的范围的起点和终点，不包含终点．如果你指定的**fromIndex**和**toIndex**相同，那么并不会删除那一个特定的元素，而是会一点作用没有，相当于并没有调用这个函数．

**AbstractList**提供了一个**subList()**方法，它接受两个参数，一个是**AbstractList**开始切分位置，另一个是**AbstractList**结束拆分的位置．

这个方法，会返回一个**SubList**，如果当前**AbstractList**实现了**RandomAccess**接口，那么就返回**RandomAccessSubList**．

那么**SubList**又是个什么鬼呢?

**AbstractList**的**subList()**方法，在形成子列表时，并不会创建一个新的**AbstractList**,并将父**AbstractList**中的相应的元素拷贝进去．那它是怎样做的呢?

它是写了一个Wrapper,这个Wrapper就是**SubList**．既然是一个Wrapper,那么它内部肯定是封装了一个**AbstractList**实例，并维护了一些其他信息．我们来看一下这个类的属性，以及其构造方法:


![](http://upload-images.jianshu.io/upload_images/4108852-da70838dacc966ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其中**l**这个**AbstractList**，就是要获取子列表的那个**AbstractList**的实例．**offset**这个属性，代表的是，从哪里开始是子列表．就是**subList()**方法的**fromIndex**参数．**size**这个属性，代表的是子列表的长度，就是**subList**的**toIndex - fromIndex**的值．

从其他的函数中，我们可以发现，实际上，修改这个**SubList**就是修改的原**AbstractList**．同时，我们也会发现，如果我们在获取到**SubList**之后，做了一些会造成**AbstractList**的**modCount**属性发生变化的操作，那么就会让**SubList**失效并抛出**ConcurrentModificationException**了．
