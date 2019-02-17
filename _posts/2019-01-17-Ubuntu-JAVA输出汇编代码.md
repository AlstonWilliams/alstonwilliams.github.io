---
layout: post
title: Ubuntu-JAVA输出汇编代码
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
有时，我们想要看看我们写的代码，对应的汇编代码到底是什么，就用到了这个．

那么如何实现呢?

在通过**java**命令运行程序时，加入**-XX:+PrintAssembly**参数即可，如果是Product版本的JVM,那么还需要在上面的那个参数前面加上**-XX:+UnlockDiagnosticVMOptions**.

比如，我使用下面的命令来运行一个测试程序:

**java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly TestTreeSet**

如果你是使用的Oracle JDK,那么很不幸，你可能会遇到这么一个错误:

**Could not load hsdis-amd64.so; library not loadable; PrintAssembly is disabled**

这咋整?

对于Linux，我们可以去[这里](https://github.com/jkubrynski/profiling/blob/master/bin/linux-hsdis-amd64.so)下载**linux-hsdis-amd64.so**,然后将其重命名为**hsdis-amd64.so**，然后移动到**$JAVA_HOME/jre/lib/amd64/**中．

然后再运行上面的命令就可以了．

我们的测试程序的源代码为:

![](http://upload-images.jianshu.io/upload_images/4108852-6062cb053581a8ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输出的汇编代码太多，这里我们只贴出一部分:


![](http://upload-images.jianshu.io/upload_images/4108852-2f1a993bc121175b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是，这只是linux版本的解决方案，如果你是使用的windows版本，那么请自行寻找解决方案．
