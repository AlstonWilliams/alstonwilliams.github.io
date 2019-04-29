---
layout: post
title: HBase的compact和rowkey全局有序(转)
date: 2019-04-29
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

最近突然想起来之前遇到的一个问题，就是rowkey的顺序不对，导致插入不到HBase中。而我们的项目中，其实一直没有关心过顺序这件事情，那为什么它不会遇到rowkey顺序不对而导致插不进去呢？

想起来之前看到的一句话，"HBase中rowkey的顺序必须是递增的"。而我们程序并没有保证这一点，那HBase是如何保证这一点呢?

从网上看到这么一段话，感觉比较真实，这里先贴下来:
> 首先当要插入个rowkey前会找到这个rowkey应该插在那个region里，这个是容易解决的，因为只要遍历所有region的endrow和startrow即可，找到新的rowkey该放在哪个region下，然后写入该region的memstore，写入memstore时会触发和memstore里原有的数据进行合并和重新排序，因为memstore里的数据有索引所以在memstore合并和重新排序是容易做到的，所以当memstore flush到文件系统后成为新的storefile也是有序的，需要注意的是新的sotorefile并不会立即和其他的storefile进行合并，所以这就会产生两个问题，1. 会造成一个rowkey可能在不同的storefile存在的情况，这是可以通过时间戳来取到最新的那个rowkey对应的数据的，2。rowkey在storefile里内部是有序的，但是在不同的storefile间是无序的，在检索一个rowkey对应的storefile这样子也是足够了。 当触发compact之后会将所有的storefile 间所有相同的rowkey进行合并，并将所有的storefile 的rowkey重新排序到一个新的更大的storefile中。以上保证了rowkey的全局有序，从以上也可以看出hbase所有的数据的更新其实都是追加。

原文地址:https://blog.csdn.net/weihongrao/article/details/17303015
