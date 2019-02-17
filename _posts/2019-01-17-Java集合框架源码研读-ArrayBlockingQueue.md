---
layout: post
title: Java集合框架源码研读-ArrayBlockingQueue
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
今天我们来介绍一个**BlockingQueue**的实现，**ArrayBlockingQueue**.

从其名字中，我们就能得知，首先，这是一个队列，其次，这个队列能够被阻塞，最后，它的底层数据结构是一个数组．

老规矩，我们还是先贴上来**ArrayBlockingQueue**的类注释:


![](http://upload-images.jianshu.io/upload_images/4108852-cbf983253aa08636.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从类注释中，我们可以发现几点我们刚才没有提到的:

- 这个队列会按照FIFO的顺序来处理数据(这也是队列的一个特性)
- 新元素被插入到队列的末尾，而要获取的元素则位于队列的首部．(这里说的队列的末尾和队列的首部只是一个逻辑概念，因为其底层的数据结构毕竟是一个数组)
- 底层数据结构数组的长度不可变
- 向一个满队列执行**put()**操作会导致线程阻塞，而向一个空队列执行**take()**操作，也会导致线程阻塞
- 可以为生产者线程和消费者线程指定一个公平策略，如果指定这样一个策略，那么总是会让等待时间最久的线程获得数据，或者插入数据
- 默认情况下，是没有上述的公平策略的，即当有多个消费者在等待时，随机选择一个线程让其获得数据，对生产者也这样
- 公平策略有好处也有坏处，好处是能够防止饥饿现象，坏处是会降低吞吐量．

## ArrayBlockingQueue的内部结构

从上文中，我们也提到了，**ArrayBlockingQueue**的内部的数据结构是一个数组．

但是，仅仅一个数组，是肯定不够的．

**ArrayBlockingQueue**中，还定义了这么几种属性:

- takeIndex: 表示下次执行**take(),poll(),peek()或remove()**操作时数据的位置
- putIndex: 表示下次执行**put(), offer(), add()**操作时，数据的位置
- count: 队列中已经存在的数据的个数

那么为什么需要**takeIndex和putIndex**这么两个字段呢?在ArrayList的实现中，当移除元素时，会涉及到数组中，数据的移动，所以，进行频繁的数据移除操作，是不适合用**ArrayList**的．同理，由于**BlockingQueue**的主要用处就是插入数据以及删除(读取)数据，如果每次删除数据时，都进行数组中数据的移动，肯定是不适合的．所以，就用这么两个指针，分别指向下次读取数据时的位置，以及插入数据时的位置．

那么，你可能就有疑问了，既然不进行数据的移动，而且数组的容量又是有限的，那么最后**takeIndex和putIndex**都指向数组的末尾啊．这咋整?

我们可以参考循环链表的形式啊．

当**takeIndex**或者**putIndex**指向数组的末尾时，就让它指向数组的起始．因为数据是按照FIFO的顺序插入以及处理的，所以，数组中，**takeIndex**前面肯定是空的．所以，我们可以按照环形的方式处理．

这样确实方便了数据的插入和删除，那么我们如何判断队列是否已经满了或者是空的呢?

我们不是有记录了数组中数据的个数的**count**属性嘛．我们可以通过这个属性来判断呀．

除了上面介绍的这几个属性，还有三个跟线程调度有关的属性．

第一个是一个**ReentrantLock**,因为我们需要保证**ArrayBlockingQueue**的每个操作都是线程安全的．

另外两个分别是**notEmpty**和**notFull**,它们的类型都是**Condition**,当执行数据读取的操作比如**poll()**,如果队列为空，那么就通过**notEmpty**让当前线程阻塞．**notFull**的用法类似.

## ArrayBlockingQueue的相关操作

**BlockingQueue**中，定义了好几个插入和访问数据的操作，这几个操作之间都有什么差别呢?就拿插入数据来说吧，有的操作是，如果队列已满，返回一个boolean值，有的是阻塞当前线程，有的是抛出一个异常．具体的请参照下表:


![](http://upload-images.jianshu.io/upload_images/4108852-a1f564a193bbed87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## BlockingQueue的使用场景

由上面的描述，相信你很容易就能看出，用BlockingQueue可以很方便的实现一个消息队列．
