---
layout: post
title: HBase-HFile解析(转)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
笔者的HBase最近出了点问题，当执行bulk load以后，再执行Get操作，会导致RegionServer陷入一个死循环中，跳不出来．进一步导致RegionServer负载过高，以及此HStore接下来的bulk load以及Compaction操作都会被卡住．因为它们都会请求同一个锁．

而死循环的部分，刚好就是读HFile的部分，所以笔者研究了一下HFile的格式．而看官方文档的时候，有的地方写的不明确，故找了一些其他的blog来看．刚好发现了一篇写的很不错的blog，不仅让我对文档中以及源码中的一些不理解的点理解了，而且对其它的方面有了更深刻的认识．故转载一下．

但是，看这篇文章以前，若你没有读过源码，或者至少看官方文档，那你，你会对这篇文章的内容不知所云．所以，建议读这篇文章以前，至少先读一下HBase中的部分代码，读一下HBase branch-1 中的`TestHFileWriterV2`这部分代码．配合这两篇文档:[Appendix E. HFile format version 2](http://hbase.apache.org/0.94/book/apes03.html)和[Apache HBase I/O – HFile](https://blog.cloudera.com/blog/2012/06/hbase-io-hfile-input-output/)．

由于原文太长，copy不易，故请点击[查看原文](https://blog.csdn.net/yulin_hu/article/details/81673695)．
