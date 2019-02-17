---
layout: post
title: Java集合框架源码研读-ConcurrentHashMap(JDK7)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
基础的集合框架，前面已经介绍的差不多了，现在我们还是介绍几个高级一些的．

首先介绍的是我们应该都熟悉的**ConcurrentHashMap**.

各位在看各种资料的时候，或多或少应该都会看到过它的身影，我曾经也看了好多次网上关于他的文章，但是并没有深入研究过．

## 注意

这里我们介绍的是JDK8之前的版本中的ConcurrentHashMap,在此之前的版本的实现方式应该都差不多，实际上，我阅读的源码是JDK6中的．而在JDK8中，源码变化很大，基本上等于重新实现了一遍．思路都不同了．

之前的很多文章在介绍ConcurrentHashMap的时候，都不会说明是哪个JDK版本中的源码，我们在读源码时，可能就有一些困惑．我刚开始读的是JDK8中的源码，然后参考IBM的一篇介绍ConcurrentHashMap的文章，就感觉有一些驴唇不对马嘴．于是Google了一下1.6版本的ConcurrentHashMap的实现，终于发现之前读的源码是错的．

但是不得不说，上面提到的那篇IBM的文章确实是不错的，我自认这篇文章不会如那篇文章那样结构清晰，通俗易懂，讲解的又深入，但是，如果我不写这篇文章记录下来最近阅读源码产生的感悟，不借着写这篇文章的时机再深入研究一点，可能学习的效果就不是很好，对吧？

所以，我会在文末附上IBM的那篇文章的链接．读者阅读到这里的时候，一定还没有被我的浅见给影响，所以，看到此处，还是请各位读者先阅读IBM的那篇文章，理解了，懂了之后，在返回此处，看看是否补充了什么．

另外，JDK8中的ConcurrentHashMap的实现，我看现在还没有人介绍．所以，接下来的这段时间，我可能会更加深入的阅读一下它的源码，给大家一个介绍．但是，在调试时，我需要用字节码操控工具ASM或者Java assist来增强一下字节码，方便我调试．可惜的是，这两款工具，我现在还并不了解．所以，这几天，我会先分别学习一下这两款工具的使用．再去调试JDK8中的ConcurrentHashMap的实现．

## 目标

在这篇文章中，我希望能够让各位对下面的问题能有一个清晰的答案:

- ConcurrentHashMap的内部结构是怎样的?
- ConcurrentHashMap为何能够保证多线程线程安全地更新数据?
- ConcurrentHashMap内部为何大量使用Unsafe类?
- ConcurrentHashMap中，写操作会阻塞读操作吗？
- ConcurrentHashMap如何保证写线程的操作能够对读线程立即可见?

## 前提条件

在各位往下读之前，我强烈建议各位了解一下Java内存模型．

这里我会简单介绍一下．

![](http://upload-images.jianshu.io/upload_images/4108852-3b6630c561c4da1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面这幅图是从**周志明**老师的**<<深入理解Java虚拟机　Java高级特性与最佳实践　第2版>>**版这本书中找到的．我认为用这幅图就能说明．

在我绘制这幅图的时候，由于不知什么原因，Dia中突然无法输入中文了，于是就先用的英文，但是也无大碍．

从上图中，我们可以看到，Java虚拟机，针对线程，将内存划分成两部分，一部分是线程独有的**Working Memory**，一部分是**Major Memory**.

如果各位对Java内存划分熟悉的话，应该会感觉这里Working Memory包括虚拟机栈和本地方法栈，Major Memory包括方法区和堆，我感觉应该是这样一种对应关系，但是周老师的书中并没有明确这么说，所以现在这只是一种猜想．

Java Thread和Working Memory以及Major Memory的关系，其实用CPU和高速缓存以及内存的关系来比喻更为恰当．这里可以把Java Thread看成CPU, Working Memory看成CPU的高速缓存，Major Memory看成我们常说的物理内存．

Java内存模型规定了所有的实例字段，静态字段以及构成数组对象的元素都存放在Major Memory中，然后线程的Working Memory中保存了该线程使用到的变量的Major Memory的拷贝，线程对变量的所有操作都必须在Working Memory中进行，而不能直接读写Major Memory内存中的变量．不同的线程之间也无法直接访问对方Working Memory内存中的变量，线程间的变量值的传递均需要通过Major Memory来完成．

这样就很清楚了，如果是一个普通变量，你对它执行了更新操作之后，由于是在Working Memory中进行的，其他的线程就无法立刻获取到这个普通变量经过更新之后的值．需要等到之后这个线程将变量回写到Major Memory然后其他线程的Working Memory中的变量失效时，才能加载到最新的数据．

那么如何解决这个问题呢?

这就需要用的volatile变量了．如果我们给变量加上这个修饰符．那么当Java线程在Working Memory中更新了这个变量之后，它还会通知其他线程，"我已经更新了变量a的值了，所以你们的Working Memory中的变量a的值就失效了，下次需要访问变量a时，记得先从Major Memory中读取呀!"．其他线程收到这个消息之后，就让它的Working Memory中的对应的变量失效，并从Major Memory加载最新的值．这样不就保证没有脏读现象了吗?对吧?

Ok,关于Java内存模型，这里就介绍这么多．IBM的那篇文章中还提到了ConcurrentHashMap中用volatile防止了指令重排序的问题．实际上，这个是由Java内存模型为volatile赋予的特殊性质．只不过我不太清楚，ConcurrentHashMap的实现跟指令重排序有什么关系．

如果你想对Java内存模型有一个更加深入的了解，强烈建议读一下**周志明**老师的**<<深入理解Java虚拟机　Java高级特性与最佳实践　第2版>>**．真的是不错的一本书，读完之后感觉对于Java的理解深入了很多．

## 内部结构

我们先来介绍一下ConcurrentHashMap的内部结构到底是怎么样的，再来介绍一下各种操作的实现．阅读一个项目的源码时，如果能了解它的内部的数据结构，它的处理流程，对于理解这个项目是大有裨益的．

对于ConcurrentHashMap的内部结构，IBM的那篇文章中，实际上已经有一个非常清晰直观的图了，这里我直接贴过来:


![](http://upload-images.jianshu.io/upload_images/4108852-a85b4e6c070427cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在**ConcurrentHashMap**中，定义了一个**Segment<K,V>[]**字段**segments**,存放了全部的**Segment**.

![](http://upload-images.jianshu.io/upload_images/4108852-c9c41c898078771e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们看一下**Segment**类的实现:


![](http://upload-images.jianshu.io/upload_images/4108852-4d0c653936b1d965.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从源码中，我们可以看到，**Segment**继承自**ReentrantLock**，这就是**ConcurrentHashMap**能够进行多线程线程安全地执行更新操作的重要原因．

同时，我们还看到，**Segment**中还有一个**HashEntry<K, V>[]**类型的变量**table**．它用于存放**Segment**中的数据．

注意上面的**table**变量是有**volatile**关键字进行修饰的，也就是说，当**table**扩容时，其他的线程是可以立即感知到的．但是，这并不意味着，**table**中的**HashEntry**也是**volatile**的．也就是说，你修改了**table**中的**HashEntry**的属性，对于其他线程来说，并不是立即可见的．

那么怎么解决这种问题呢?即如何解决数组是volatile而数组中的元素不是volatile的问题呢?

第一种,也是最容易想到的，就是，我们把数组中的元素中的能够被修改的字段，也给弄成volatile的，不就得了嘛！但是除了这样做以外，我们还需要配合使用**Unsafe**类的**putOrderedObject**(还有一个方法，putObjectVolatile，我感觉也可以，但是没有验证)方法，触发Working Memory变量失效的过程．**Segment**中就是采用的这种方式．

第二种方案，就是，修改了数组中的元素的时候，将这个数组重新赋给旧数组，用代码表示就是:

~~~~
  table[0].value = 1;
  table = table;
~~~~

这个应该也很容易理解吧?因为**table**就是**volatile**,你对它重新赋值一遍，当然也会触发上面我们提到的那个告诉其他线程**Working Memory**失效并从**Major Memory**中重新加载的过程．

这样我们就能在数组中的元素更新了之后，对于其他线程来说，也是立即可见的．

**HashEntry**是一个链表类型的数据类型，当发生冲突时，采用链接法来处理冲突:


![](http://upload-images.jianshu.io/upload_images/4108852-1530f32afe86b13e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里注意**value**和**next**都用**volatile**关键字进行修饰了．

有没有感觉**Segment**除了继承自**ReentrantLock**之外，其他的地方都很像一个**HashMap**?内部也是采用数组作为底层数据结构，冲突时也是采用链接法来解决，当链表上的冲突的元素太多了，也会进行树化，当整个超过了**threshold**，也会扩容并进行重新哈希．

所以，我感觉其实**ConcurrentHashMap**本质上就是将多个由**ReentrantLock**守护的**HashMap**组合成了一个更大的**HashMap**而已．

在**ConcurrentHashMap**中，还有很多很重要的参数，比如**DEFAULT_CONCURRENCY_LEVEL**，默认为**16**，表示默认情况下，一个**ConcurrentHashMap**包含**16**个**Segment**.还有代表整个**ConcurrentHashMap**的默认最小容量的**DEFAULT_INITIAL_CAPACITY**,代表整个**ConcurrentHashMap**的默认负载因子的**DEFAULT_LOAD_FACTOR**.

## 构造方法

由于ConcurrentHashMap的一些元数据比如**Segment**的数量都是在构造方法内进行初始化的，所以如果我们了解了构造方法，对于这些变量的作用会有一个更加清晰的认识．

![](http://upload-images.jianshu.io/upload_images/4108852-954dec8481f6e64d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-14dd5217520cfd3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实这里，你只要拿默认值往里面一带，走一遍，就清楚了．

咱们先用默认值走一遍，默认情况下，传入的参数分别是**ConcurrentHashMap.DEFAULT_INITIAL_CAPACITY, ConcurrentHashMap.LOAD_FACTOR, ConcurrentHashMap.CONCURRENCY_LEVEL**：

~~~

initialCapacity = DEFAULT_INITIAL_CAPACITY = 16
loadFactor = DEFAULT_LOAD_FACTOR = 0.75
concurrencyLevel = DEFAULT_CONCURRENCY_LEVEL = 16
MIN_SEGMENT_TABLE_CAPACITY = 2

sshift = 0 -> 4
ssize = 1 -> 16
segmentShift = 32 - sshift = 32 - 4 = 28
segmentMask = ssize - 1 = 15
c = initialCapacity / ssize = 16 / 16 = 1 -> 4
cap = 2 -> 4

~~~

其中上面的那一部分是传入的参数以及默认值，下面的那一部分是在这些默认值的情况下，各个局部变量的变化情况．**a->b**表示变量的值经过一个循环以后由**a**变成了**b**.

这个应该没有困难的吧．实在不理解，自己动手验算一下便是．

从源代码中，我们能够看到，**Segment**的数量由**ssize**变量确定，**Segment中table**的大小由**cap**变量确定．


![](http://upload-images.jianshu.io/upload_images/4108852-9a1a681d4f1d276f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面这段代码我们可以看到，如果**ssize**小于**concurrencyLevel**，那么**ssize**便会一直变化为两倍大小．也就是说，如果我们指定**concurrencyLevel**为17,那么实际上会有32个**Segment**.

从这也说明了，**concurrencyLevel**参数，并不是指定的**Segment**的数量．这个参数只是说，你估计一下会有多少个线程并发访问这个**ConcurrentHashMap**,然后**ConcurrentHashMap**根据你估计的这个值，确定有多少个**Segment**.

好，现在我们知道了**ssize**代表**Segment**的数量了，那么**sshift**是什么含义呢？

它代表的是ssize的左移偏移量．

**A的左移偏移量表示，要把1左移多少位才能得到A**.打个比方，int型的16的二进制形式为**0000 0000 0000 0000 0000 0000 0001 0000**.很明显，16的左移偏移量为4.

左移偏移量有什么作用呢?


![](http://upload-images.jianshu.io/upload_images/4108852-8f83b088c52a7772.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

请看这两行，就用到了**sshift**这个左移偏移量．那么**segmentShift和segmentMask**这两个**ConcurrentHashMap**的实例变量是干嘛的呢?相信你能够猜出来，当然是用这两个实例变量定位**Segment**的呀．


![](http://upload-images.jianshu.io/upload_images/4108852-5a5ba31d5759085b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个方法用于根据hash获取对应的**Segment**．如果各位不了解**Unsafe**类，那么可能不太能理解这个方法．我简单解释一下．

**ConcurrentHashMap**中，获取数组中的元素时，都不是使用的**array[index]**的形式，而是直接从内存中获取，采用这样的公式来定位内存中，这个元素的位置:数组在内存中的起始地址 + (元素在数组中的索引 * 每个元素的尺寸)．为什么采用这种方式呢?因为在Java中，对数组通过索引获取元素时，需要检查有没有范围越界的问题，没有采用直接访问内存的这种方式的效率高．在ConcurrentHashMap中，为了进一步提高效率，就采用了直接从内存访问的方式．

上面的源代码中，**(h >>> segmentShift) & segmentMask**就是用来获取哈希值为h的**Segment**在**segments**数组中的索引，其实源码上采用的方法就是咱们上面说的那个公式，但是源码中用位操作改进了一下，后面我们会介绍．

咱们还是回到那个问题上，**sshift**这个左移偏移量到底有什么用?

我们可以看到，由于**segmentMask**的值为15,所以其有效位数就为低四位，所以，我们必须取**h**的高四位来进行运算，而int型又是32的，所以，**segmentShift**需要是28，才能取得**h**的高四位．你看**segmentShift**为28，**sshift**为4，这两个有什么关系?

它们加起来正好是32,正好是int型数据的位数!

所以，左移偏移量sshift也可以说是我们希望int型数据经过无符号右移操作之后，可以取得的最高sshift位．所以，为了取得这最高的sshift位，我们需要将int型数据右移**32-sshift=segmentShift**位．

那上面说的公式的优化是怎么回事?

我们先看一下**SSHIFT**这个实例变量是怎么得到的，从**ConcurrentHashMap**的底部可以得到:

![](http://upload-images.jianshu.io/upload_images/4108852-4e65005c0ab68060.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，**ss**变量表示的是**Segment[]**数组中，增量的长度，也就是说，数组中相邻的两个索引在内存地址上的距离，那么，**31 - Integer.numberOfLeadingZero(ss)**表示什么意思呢?

我们查看Integer中的**numberOfLeadingZeros**方法的源码:


![](http://upload-images.jianshu.io/upload_images/4108852-a6428ece72cc956c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我们并不深入这个方法的实现，而是看它的注释，从注释中，我们很清晰的就能发现，有这么一个公式:


![](http://upload-images.jianshu.io/upload_images/4108852-15a2ab3d0fbec0eb.gif?imageMogr2/auto-orient/strip)

其中**floor**的意思是，小于等于并且最接近以二为低的x的对数．

在这里，由于ss是４(通过在ConcurrentHashMap中加一条打印语句来查看．因为在32bit机器上或者最大堆内存小于32Gb的机器上，一个对象的引用占4个字节，所以ss是４)，所以，上面的公式就等于:


![](http://upload-images.jianshu.io/upload_images/4108852-b5ce9a2ae2c469a0.gif?imageMogr2/auto-orient/strip)

所以，我们有:

![](http://upload-images.jianshu.io/upload_images/4108852-1488af918a5d5282.gif?imageMogr2/auto-orient/strip)

在结合**(index << SSHIFT) + SBASE**，我们可以得到下面的式子:


![](http://upload-images.jianshu.io/upload_images/4108852-7e821c8b7922bfb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中index就是哈希为**h**的那个**Segment**在**segments[]**数组中的索引，**SBASE**这个变量表示的是，**segment[]**数组在内存中的其实地址．

你看，最后得到的这个式子不就是我们之前说的那个没有经过优化的式子吗？

实际上，在明白**(index << SSHIFT) + SBASE**是一个经过优化之后得到的公式之后，我就对ConcurrentHashMap的作者敬佩不已．我们可以看到，尽管优化之后的式子需要进行更多的计算，但是更多的是位操作，再加上一次减法操作，这些都是机器能够直接给我们提供的指令能做的运算，都是逻辑计算单元直接能做的操作，不像乘法这种操作．

这个构造函数下面的内容就很简单了吧，无非就是确定一下**Segment**中**table**的长度．

## put(K key, V value)方法和get(K key)方法

如果前面的内容，各位已经了解的话，下面的这几个方法，就都不是事了，就特别特别简单了！

这两个方法应该是我们最常用的两个方法，现在我们就来介绍这两个方法．

#### put(K key, V value)方法

![](http://upload-images.jianshu.io/upload_images/4108852-9ce92d67906abd33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从put()方法中，我们可以看到，先是对**key**进行两次哈希运算，然后找到对应的**Segment**，如果不存在那个**Segment**，就创建它，并调用**Segment**的**put(K key, int hash, V value, boolean onlyIfAbsent)**方法，将数据插入．

从**ConcurrentHashMap**的**put()**方法中，我们可以看到，是没有任何锁操作的，所以对**ConcurrentHashMap**进行数据插入的操作时，是不会锁住整个**ConcurrentHashMap**的．

接下来，我们看一下**Segment**的put()方法．


![](http://upload-images.jianshu.io/upload_images/4108852-14e29243d2ff4d0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-ebb9b5cc088467ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个方法中，我们可以看到，一上来就是获取锁，最后再释放锁，所以，每次对**ConcurrentHashMap**时，实际上只会锁住一个**Segment**.

同时，从第二章图片中，我们还可以看到，如果发生冲突了，总是会在链表的前面插入新的节点，而不是在链表的末尾插入节点．

引用IBM的那篇文章中的图片，就是:

![](http://upload-images.jianshu.io/upload_images/4108852-abdc119cf50ba0ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是说，链表中节点的顺序，实际上是跟插入的顺序相反的．

#### get()方法

get()方法更加简单．

![](http://upload-images.jianshu.io/upload_images/4108852-c0ae70ab9c9d27fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还是先计算两次哈希，然后先根据hash找到对应的**Segment**,然后在从**Segment**中的**table**中寻找，如果是个链表，就采用顺序查找的方式进行查找．

## 总结

我们可以看到，ConcurrentHashMap需要两次计算Hash，这个效率肯定是比不上只计算一次Hash的，所以，如果不是在并发情况下，就不要使用这个类．

另外，ConcurrentHashMap和Hashtable以及由Collections.synchronizedMap方法包装而成的线程安全的Map有什么区别?

ConcurrentHashMap相对于其他两者，有点是性能更好，能够允许多个线程并行执行更新操作，当然前提是这两个线程不能是更新同一个**Segment**中的数据，并且写线程不会阻塞读线程．而其他两者，由于都是使用synchronized来进行同步，所以其实际上只能串行地更新和读取数据．并且写线程会阻塞读线程．

但是，ConcurrentHashMap相对于其他两者，当然也是有弱点的，就是它实现的是弱一致性，对于某些需要强一致性的场景中，用ConcurrentHashMap并不适合．

## 参考文章

[探索ConcurrentHashMap高并发的实现方式](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/index.html)
