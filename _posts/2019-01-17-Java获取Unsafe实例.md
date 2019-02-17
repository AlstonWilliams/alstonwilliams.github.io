---
layout: post
title: Java获取Unsafe实例
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
**Unsafe**是JDK中的一个内置的类，用于直接根据内存地址访问元素．它也提供了很多好用的方法，比如，用volatile的方式设置数组中的元素．

但是，这个类的作者，不希望我们使用它，因为我们虽然我们获取到了对底层的控制权，但是也增大了风险，安全性正是Java相对于C++/C的优势．

这个类，默认情况下，只能被由**BootstrapClassLoader**加载器加载的类所使用.

那我们要是想使用的话，该如何来获取呢?

通过下面的几行代码即可获得一个**Unsafe**的实例:

~~~~
Field singleoneInstanceField = Unsafe.class.getDeclaredField("theUnsafe");

singleoneInstanceField.setAccessible(true);

Unsafe unsafe = (Unsafe)singleoneInstanceField.get(null);
~~~~
