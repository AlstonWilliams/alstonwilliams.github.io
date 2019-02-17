---
layout: post
title: C语言中,char--pointer="hello"和char-pointer[]="hello"之间有什么区别-
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- C语言
tags:
- C语言
---
我们在学习C语言时,一般会觉得指针和数组没有本质的区别,然后滥用就会造成一些看起来莫名其妙的错误.

在我写一个将字符串转换成大写的形式的函数时,遇到了**Segment Fault**错误.上图:


![](http://upload-images.jianshu.io/upload_images/4108852-f7ac27f1e1ee0eb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


开始以为是非法访问内存造成的,然而调试时,并没有发现引用不存在的内存这种情况出现.

要理解这个问题,我们首先需要理解,C程序在运行时,是如何使用内存的.

我找到了这么一张图片:


![](http://upload-images.jianshu.io/upload_images/4108852-dd4d93855353e38e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里我们主要关注**initialized data**这个区域.

在这个区域中,包含了程序员初始化的变量.这个区域又被进一步划分成只读区和读写区.

当我们使用**char pointer[] = "hello"**时,它会被存储到读写区中.而当我们使用**char *pointer = "hello"**时,**"hello"**会被存储到只读区,而**pointer**这个指针会被存储到读写区.所以,我们使用指针修改只读区的时候,因为是**undefined operation**,所以会出现**Segment Fault**的异常.

关于C程序内存使用的更多说明,请看下面这个链接:
http://www.geeksforgeeks.org/memory-layout-of-c-program/
