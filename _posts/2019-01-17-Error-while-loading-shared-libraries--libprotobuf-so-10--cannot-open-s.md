---
layout: post
title: Error-while-loading-shared-libraries--libprotobuf-so-10--cannot-open-s
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Protocol-Buffer
tags:
- Protocol-Buffer
---
在安装protobuf的c++版本的编译器时,使用**protoc --version**命令查看版本时,遇到这个问题.

解决方法为,安装protobuf的库,我这里因为是c++版本,所以安装的是**libprotobuf-c0**.使用下面的命令安装:
**sudo apt-get install libprotobuf-c0**

在Ubuntu环境下可用.
