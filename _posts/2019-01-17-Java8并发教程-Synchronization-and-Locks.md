---
layout: post
title: Java8并发教程-Synchronization-and-Locks
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
这是该系列教程的第二篇.其中用到了一个工具类,和其中的两个方法.如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-7aaa30b6d1ab476b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## Synchronized

在第一篇教程中,我们已经介绍了如何通过** Executor Service**来并行执行任务.但是,这也引入了一个新的问题.即我们如何并发的访问那些共享的变量.假设我们打算用多个线程来并发地增加一个数字.我们使用下面的代码:


![](http://upload-images.jianshu.io/upload_images/4108852-ee8444f82970bc96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![](http://upload-images.jianshu.io/upload_images/4108852-496a40575630d27f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到,其结果不是正确的结果,即10000.那为什么会出现这种情况呢?这是因为我们不恰当的让多个线程并发的访问并设置共享变量而造成的.本例中,共享变量即为** count**.

当执行一个加法操作时,需要按三步进行:1.读取当前值2.使当前值加一3.将这个新值写入到变量中.如果有两个线程同时执行第一步,即读取当前值,它们获得了相同的值,就会造成写丢失.也就是说,第二个线程在执行第三步时,会覆盖掉第一个进程写入的结果.

Java中,我们可以通过使用** synchronized**关键字,来防止这种情况的出现.

我们使用**synchronized**关键字来改写上面的执行加法操作的函数:


![](http://upload-images.jianshu.io/upload_images/4108852-f00f94c5b9a5c393.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


现在我们让线程通过执行这个函数来并发地进行加法操作:


![](http://upload-images.jianshu.io/upload_images/4108852-65f70905ae41534e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


现在我们可以看到,结果是正确的.不管你执行多少次,结果都是正确的.

** synchronized**关键字,不仅可以用在函数上,还可以用在代码块中:


![](http://upload-images.jianshu.io/upload_images/4108852-b4766f30f7856f7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在** synchronized**关键字的内部, Java使用一个叫做** monitor**的东西来管理它,** monitor**也被称作** monitor lock 或 intrinsic lock**. monitor是和对象关联在一起的,每个** synchronized**的方法,针对一个对象,都使用同一个**monitor**.

** synchronized**也有可重入的特性,也就是说,即使当前线程已经占有了锁,它还是可以请求相同的锁的,这就避免了死锁的产生.

## Locks

除了通过使用** synchronized**这种隐式锁来进行同步,Concurrency API中,还提供了大量显式锁.通过使用这些显式锁,我们能对并发进行更好的控制.

下面我们会一个个的介绍这些显式锁.

### ReentrantLock

这个锁是一个互斥锁.它也实现了** synchronized**中的隐式锁的基本特性,当然,它还是有一些自己的特性的.这个锁也是可重入的.

我们使用** ReentrantLock**来实现上面的例子:


![](http://upload-images.jianshu.io/upload_images/4108852-ca4855f1724dde37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们通过** lock()**方法来获得锁,而通过** unlock()**方法,来释放锁.我们要用try/catch来包装我们的代码,来防止锁得不到释放.这个函数也是线程安全的.如果其他线程在这个锁没有释放之前,想要获得锁,就会被阻塞.只有一个线程能够同时占有锁.

除此之外,锁还提供了其他的函数,如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-43d5e9bab41dc7ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


第一个任务获得锁,然后暂停一秒钟.而第二个任务则获取锁的当前状态,并将其输出出来.


![](http://upload-images.jianshu.io/upload_images/4108852-d03165425aa894da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** tryLock()**会尝试获取锁,而不会像** lock()**方法一样,阻塞线程.我们需要在执行那些需要访问共享变量的操作之前,检查一下其返回值,以防止不同步的情况.

### ReadWriteLock

** ReadWriteLock**包含一对锁,分别用于对共享变量的读和写操作.** ReadWriteLock**背后的原理是,如果当前没有线程来修改共享变量,那么允许多个线程来访问共享变量,而没有什么危险.所以说,当没有线程持有写锁的时候,可以有多个线程持有读锁.这在那些读操作远大于写操作的场景中,极大的提高了性能和吞吐量.


![](http://upload-images.jianshu.io/upload_images/4108852-8691a4eb9d8736aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![](http://upload-images.jianshu.io/upload_images/4108852-2cebc837273fde7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在你执行上面的代码时,你会注意到,只有当写锁被释放之后,后面的线程才会同时获取到读锁.而不需要等第一个线程的读锁释放之后,第二个线程才能获取读锁.

### StampedLock

Java8中,还增加了一种叫做** StampedLock**的锁,这个锁也有读锁和写锁,就跟上面的** ReadWriteLock**一样.但是,和** ReadWriteLock**不同,它会返回一个** long**类型的值,我们可以通过这个值来释放锁,检查锁是否有效.另外,它还包含一种叫做乐观锁的锁.

我们有下面的代码:


![](http://upload-images.jianshu.io/upload_images/4108852-b3d767401e9dbcd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们通过** readLock()**方法来获取读锁,通过** writeLock()**方法来获取写锁.需要注意的是,** StampedLock**并没有可重入的特性.如果没有锁可用,则调用上面的方法,会导致返回一个值,并阻塞线程,即使当前线程已经有锁了.所以你使用这个锁的时候,要小心,别出现死锁的情况.

跟** ReadWriteLock**锁一样,要获得读锁,必须等待写锁被释放.

我们使用下面的这个例子,来了解乐观锁:


![](http://upload-images.jianshu.io/upload_images/4108852-a906e86118b1adb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


** tryOptimisticRead()**方法,会获得一个乐观读锁.它总会返回一个值,而不会阻塞当前线程.如果有线程持有写锁,则返回值是0.所以,我们需要通过** lock.validate()**方法来检查一下返回值,来确定是否真的有读锁可用.

上面的代码,输出如下:


![](http://upload-images.jianshu.io/upload_images/4108852-e138093cc3a9e436.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们应当在获取到乐观锁之后,就立即使用它.因为它随时可能无效.与平常的读锁不同,乐观锁不会阻止其他的线程获得写锁.也就是说,在一个线程占有乐观锁的时候,其他的线程还是可以获取到写锁的,而不需要等待乐观锁被释放.当其他线程获得了写锁之后,乐观锁就失效了.即使那个线程后来又释放了写锁.

所以,如果使用乐观锁的话,我们需要时刻验证乐观锁是否还有效.特别是在执行对共享变量的写操作之前.

有时,我们需要将一个读锁转换成写锁.** StampedLock**提供了** tryConvertToWriteLock()**这个方法,来进行转换.


![](http://upload-images.jianshu.io/upload_images/4108852-ab48cb7bccfe2365.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面的那个任务,会先获取读锁,并尝试输出count变量的值.但是当其等于0时,它会将读锁转换成写锁,如果转换成功,则将23赋给count.如果不成功,则直接通过** lock.writeLock()**来以阻塞的方式获取写锁.然后将23赋给count.最后,再释放锁.

## Semaphores

除了锁,Concurrency API也支持信号量.锁通常用于互斥的访问某些资源,而信号量用于限制一定的线程可以同时访问某些资源.

下面这个例子,演示了如何使用信号量:


![](http://upload-images.jianshu.io/upload_images/4108852-35483984392db9eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Executor允许同时执行十个线程,但是因为我们将信号量设置成了5.所以实际上,只有5个线程有机会来执行操作.

输出如下:


![](http://upload-images.jianshu.io/upload_images/4108852-e5c7af8a12fff8c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


只有前五个线程,能够获得一个信号量,然后来执行暂停五分钟的操作.其他的线程,因为没有信号量可以获取了,就只能在控制台打印处** Could not acquire semaphore**.
