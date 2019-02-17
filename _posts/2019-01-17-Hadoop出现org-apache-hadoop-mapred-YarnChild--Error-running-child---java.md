---
layout: post
title: Hadoop出现org-apache-hadoop-mapred-YarnChild--Error-running-child---java
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
出现这个错误的原因，最直观的就是，确实是堆内存不够。

而我出现这个问题的原因是这样的：输入文件是SeqenceFile，但是在Job中并没有设置输入的格式为SequenceFile，就出现了这个错误。
