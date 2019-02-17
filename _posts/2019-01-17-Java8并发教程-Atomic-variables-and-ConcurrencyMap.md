---
layout: post
title: Java8并发教程-Atomic-variables-and-ConcurrencyMap
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
这是本教程的最后一篇.我们还是用到了上一篇中提到的那个工具类和其中的两个方法.请看上篇文章,来获取此代码.

## AutomicInteger
** java.concurrent.atomic**包中,提供了大量的有用的类,来执行原子性的操作.原子性的意思是,所有的操作要不就都执行成功,要不就都不执行成功.

这些原子性的类内部,都使用了著名的** CAS**指令.现代处理器都支持这条指令.相对于使用锁的** synchronized**关键字来说,这条指令操作起来更快.所以,如果我们只想并发的对一个共享变量进行操作,那使用这些原子性的类,更为方便和快捷.

我们首先看一下** AutomicInteger**这个类如何使用:


![](http://upload-images.jianshu.io/upload_images/4108852-4c6ec1d4bef0cc63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


通过使用** AutomicInteger**类型,而不是** Integer**类型,我们能够以线程安全的方式,来并发的修改一个数值.而不需要跟以前一样,使用** synchronized**关键字或者显式锁.** AutomicInteger**类型提供的** incrementAndGet()**方法,是线程安全的,所以我们可以放心的在多个线程中使用这个方法,而不需要担心那些预料之外的事情发生.

** AutomicInteger**类型还提供了很多其他的原子性的方法.其中**updateAndGet()**方法允许我们传入一个lambda表达式,来对此数值就行操作.


![](http://upload-images.jianshu.io/upload_images/4108852-fd772e87b2c83438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** accumulateAndGet()**方法,接受一种** IntBinaryOperator**类型的lambda表达式.在下面这个例子中,我们使用这个方法,来并发的计算0到1000的和.


![](http://upload-images.jianshu.io/upload_images/4108852-4a38ae38595a48cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** java.concurrent.atomic**包中,还提供了很多其他的原子类.如** AutomicBoolean, AutomicLong, AutomicReference**等.

## LongAddr

** LongAddr**和** AtomicLong**相似,用于连续的为数值增加一个值.


![](http://upload-images.jianshu.io/upload_images/4108852-48c07ff44eeab785.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** LongAddr**还提供了其他的原子性的,线程安全的函数,比如** add()**,和** increment()**.但是,这些方法并不只是单纯的计算一个结果,它还在内部维护了很多变量,用于减少线程之间的冲突.我们可以通过** sum()**方法和** sumThenRest()**方法,来获取计算的结果.

这个类,在执行写操作的线程多于执行读操作的线程这种情景中,比较常用.通常用在那些需要获取统计数据的情况中.比如,你想要计算服务器接受的请求数.** LongAddr**的缺点是,由于要在内存中维护大量的变量,所以它比较耗内存.

## LongAccumulator

** LongAccumulator**和** LongAddr**相似,但是更加常用.它执行的不是简单的操作,它接受的是** LongBinaryOperator**类型的lambda表达式.如下例所示:


![](http://upload-images.jianshu.io/upload_images/4108852-95327b52cfdfb8aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们通过函数** 2 * x + y**和初始值1,创建了一个** LongAccumulator**.每次调用** accumulate(i)**函数,当前值会作为lambda表达式的x值, i会作为lambda表达式的y值,传递到lambda表达式中.

** LongAccumulator**就像** LongAddr**一样,内部也维护了大量的变量,来减少线程之间的冲突.

## ConcurrentMap

** ConcurrentMap**接口,扩充了** Map**接口,成为了并发编程中,最有用的一个接口.

我们想创建一个包含四对数据的** CouncurrentMap**.在后面我们将用它来实验那些函数.


![](http://upload-images.jianshu.io/upload_images/4108852-16e8375a1c9ec938.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** forEach()**方法接受一个类型为** BiConsumer**的lambda表达式作为参数,其中此lambda表达式的参数为map中的键值对.我们可以用** forEach()**这个方法来在当前线程内,串行的迭代这个map.


![](http://upload-images.jianshu.io/upload_images/4108852-73c38ab30974cd9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** putIfAbsent()**方法,会在给定的key没有value时,为其添加一个value.至少在** ConcurrentHashMap**中,其实现是线程安全的.


![](http://upload-images.jianshu.io/upload_images/4108852-dfee53fb5cd16dc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** getOrDefault()**方法,会尝试获取给定key的value,如果不存在,则返回我们指定的默认值.


![](http://upload-images.jianshu.io/upload_images/4108852-edc237bc1f057d5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** replaceAll()**方法,用于替换此Map中,满足条件的项的value.


![](http://upload-images.jianshu.io/upload_images/4108852-a3ff3151294e5ad3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** compute()**方法,允许我们对特定的项进行转换.


![](http://upload-images.jianshu.io/upload_images/4108852-3eee4150b73f73c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


除了** compute()**方法,还有两个变体,** computeIfAbsent()**和** computeIfPresent()**,分别在给定的key不存在时和存在时进行操作.

** merge()**方法,用于对给定key的value进行操作,生成一个新的值.


![](http://upload-images.jianshu.io/upload_images/4108852-2bf1e3e4653ce8a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## ConcurrentHashMap

上面介绍的函数,都是** ConcurrentMap**这个接口提供的.这些函数可以被任何实现了** ConcurrentMap**的类使用.除此之外,** ConcurrentHashMap**还提供了很多其他用于并发操作的函数.

就像parallel streams一样,这些函数内部都使用** ForkJoinPool**,在Java8中,我们可以通过** ForkJoinPool.commonPool()**函数来获得一个** ForkJoinPool**.这个线程池,默认可以使用的线程数,取决于你的机器上的CPU上,有几个核心.在我的四核的机器上,其为3.


![](http://upload-images.jianshu.io/upload_images/4108852-7469816ffbae04d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以通过设置JVM的参数,来修改这个数值.


![](http://upload-images.jianshu.io/upload_images/4108852-1f4ae735b23f7f16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们还是使用上面的那个包含四条数据的map.但是这里我们不使用** ConcurrentMap**这个接口了,而是使用** ConcurrentHashMap**这个具体实现类,来使用** ConcurrentHashMap**中特有的函数.


![](http://upload-images.jianshu.io/upload_images/4108852-2895812a7cc955ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Java8中,提供了三种并行操作的函数:** forEach()**, ** search()**和** reduce()**.这些函数的第一个参数,都是如果要启动并发执行的话,Collection的最小阈值.比如,如果我们设置了这个阈值为500,而map的大小为499,那就会在一个线程中,串行的执行.而如果map的大小大于500,就会开启多个线程,并行的执行.在后面的例子中,我们将这个阈值设置为1,这就意味着,总是并行的执行操作.

### ForEach

** forEach()**方法,用于并行的执行迭代map中的key/value对的操作.因为在我的机器上,** ForkJoinPool**的最大尺寸为3,所以在下面的例子中,你会看到,最多启动了三个线程.


![](http://upload-images.jianshu.io/upload_images/4108852-48c64c968d362409.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Search

** search()**函数用于并行的查找map中给定的key的值,如果找到,就返回其value,如果找不到,就返回null.如果会找到多个,则其返回值不确定.也要注意,ConcurrentHashMap中的元素是无序的.


![](http://upload-images.jianshu.io/upload_images/4108852-e1f47a2ddd21307f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


还有一个只用于搜索map的value的方法,如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-31f1f1887f1ae1f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Reduce

** reduce()**函数,接受两个类型为** BigFunction**的lambda表达式.第一个lambda表达式,会将map中的每一个key/value对转换成一个值,然后第二个lambda表达式,会将这些转换后的值,拼接成一个单一的结果.它会忽略** null**.


![](http://upload-images.jianshu.io/upload_images/4108852-ccaf26be39d5bcbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
