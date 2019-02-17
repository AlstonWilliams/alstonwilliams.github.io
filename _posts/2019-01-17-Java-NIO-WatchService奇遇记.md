---
layout: post
title: Java-NIO-WatchService奇遇记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
在扩展Apache Flume的Taildir Source的过程中，由于感觉其采用IO轮询的方式，而不是Java NIO，会有性能问题，于是就打算通过Java NIO将相关的部分重写一遍．

我们的想法是这样的，先监控某个目录，然后当有文件修改事件触发时，判断一下被修改的文件是否是某些特定的文件，如果是，则读取其新增的内容，并发送给Channel.

通过Google,我们找到了WatchService这个东西，然后参考官方的源代码实现了一遍．

开始没有什么问题．实际上是有问题，但是我没有注意到．

用WatchService实现的代码如下:


![](http://upload-images.jianshu.io/upload_images/4108852-eb71d0abe9d89265.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-06e2199886c91ce3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-2e64cd98940470d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，其中有一个死循环，不断地读取新的事件，判断是否是我们要监控的文件，如果是，则读取新的内容并发送给Channel.

问题是，它是死循环啊!!!

昨天下午在系统的其他部分时，由于扩展后的Flume也是其中的一个必须的组件，所以就一直开着Flume.期间，我就纳闷，怎么电量这么不耐用呢?之前都是能够撑两个小时的，怎么这次只用不到一个小时就没电了?

用top命令查看了一下，发现一个Java应用的CPU利用率一直都是100%+.

用ps命令查看了一下对应的进程，发现就是我们的Flume进程．

此时，我恍然大悟．想到了可能是死循环导致的CPU利用率太高．

这时，想起了自己在改写这部分时的初衷，是希望能够有这么一个接口，我们可以传入要监听的事件，然后传入一个回调函数．当触发相应的事件时，就调用回调函数．

而WatchService显然不是我需要的．

再次Google了一下之后，发现有人推荐用一款叫JNotifier的库．从StackOverflow上得知，这个库需要调用一个本地库，而这个本地库是封装了Linux上的某个调用，用C语言封装的，打包成了一个so文件．

所以，我们想用的话，就得先将这个so文件下载下来，然后加入**java.native.path**中．开始我是拒绝的，但是看到它的API那么简单，楚楚动人，我就忍不住想要尝试一下．然而，官网上并没有明确说明我们要如何来做．

结合出现的问题，StackOverflow了好长时间，用他们提供的方法都没有解决．直到发现有一个人提问说他在Java 1.7和Java 1.8下都尝试过，但是都出现那个错误，底下有人回答说，Java 1.8就不行．看到这，我终于失去了耐心！我用的就是Java 1.8．

最终，我删除了一切跟JNotify相关的内容，然后重新踏上了寻找之路．

后来，我发现了一个库，叫Apache Commons VFS.毕竟是Apache的产品，用起来更放心一些．使用起来，发现它的API甚至更加简单，最终实现的代码如下:


![](http://upload-images.jianshu.io/upload_images/4108852-2e3c66a8e291536f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-9defb86d0c6dde26.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-ba43b04fdfd5a024.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-c259a7e4ced053fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在，Flume的CPU利用率就保持正常了．

但是，Apache Commons VFS的具体实现，我并没有查看．我不知道它到底是如何实现的，内部是否还是通过轮询的方式实现的．虽然它现在能正常工作，但是我们并没有对它进行过深度测试，所以，实际上，我们是不能信任它的．我们并不能认为它就完美无暇．因为它现在对我们就是一个黑匣子!
