---
layout: post
title: IntelliJ-IDEA中导入ZooKeeper源码，但是无法导航到其他类
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
在开始阅读ZooKeeper的源码的过程中，发现无法导航到其他的类，甚至就连JDK自带的那些类都不能导航进去．于是Google了一下，结合了一下StackOverflow上的答案，解决了这个问题．

首先，我们在下载ZooKeeper的源码之后，可以看到其有如下几个直接子目录:


![](http://upload-images.jianshu.io/upload_images/4108852-100b1193249e1f7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其中，源码是存储在**src**目录下的．我们进一步查看**src**目录，可以看到其下面包含了**java**文件夹，其目录结构为:


![](http://upload-images.jianshu.io/upload_images/4108852-69f1d0be123fbf9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我这里只是列出了三级目录的结构．

我们可以看到，**java**文件夹中，又包含了**main**文件夹．其中**main**文件夹中包含了全部的主要源码．

那么，我们为了让IntelliJ IDEA能够找到源码，就需要将**main**这个目录设置为**Source Root**. **Source Root**是IntelliJ IDEA中的一个概念，设置方式如下:


![](http://upload-images.jianshu.io/upload_images/4108852-86e8f61266f3a0e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这样设置好之后，就能导航到其他类了．

但是，此时你很可能会遇到找不着其他类的错误，比如**org.apache.zookeeper.data**包．

那么为什么找不到呢？因为这些包是ZooKeeper通过Jute生成的．

还有一些ZooKeeper的依赖包，放到其他的目录中，我们没有将它们添加到**CLASSPATH**中．那么如何来解决呢?

第二种情况容易解决，而第一种就比较麻烦了．我反复找了好久都没有找到它们生成的代码到底在哪，最后通过查看ZooKeeper根目录下的**zookeeper-3.4.9.jar**，发现原来在这里面．

所以，我的解决方法是，将ZooKeeper根目录下的**lib**目录，根目录下的**src/java/lib**目录以及根目录下的**zookeeper-3.4.9.jar**添加到项目的依赖中．

具体操作方法为:

- 打开**File -> Project Structure**:


![](http://upload-images.jianshu.io/upload_images/4108852-312b5f96b839a8c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 
- 在**Modules**的右侧的面板中，找到**Dependencies**，在其中添加上面说的那几个依赖:

![](http://upload-images.jianshu.io/upload_images/4108852-fd7e7a35f1cd1c21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
这样就行了．
