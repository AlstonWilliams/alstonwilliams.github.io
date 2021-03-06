---
layout: post
title: 编译ZooKeeper
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
最近在研究分布式系统，由于ZooKeeper作为一个键值存储系统，结构相对比较简单，但是麻雀虽小五脏俱全，是一个不错的适合新手阅读的项目，所以就选择了ZooKeeper.

在研究源码的过程中，我们少不了要自己进行调试．所以我们首先需要会编译ZooKeeper.

其实编译过程很简单．ZooKeeper使用了Ant+ivy作为依赖管理系统以及构建系统，其中ivy作为依赖管理系统，Ant作为构建系统．所以，我们需要先在本机上安装Ant+ivy.

那么如何安装Ant呢？去官网下载最新版的Ant构建好的包，解压并设置**ANT_HOME**，然后把**${ANT_HOME}/bin**添加到**PATH**环境变量下．过程很简单，很多JAVA工具都是这么一个安装过程．各位应该对其不陌生．

接下来就需要安装ivy了．安装ivy就更加简单了．去官网上下载对应的包，然后将里面的**ivy-version.jar**复制到**${ANT_HOME}/lib**目录下即可．

如果你在安装ivy之前，先读了其文档，那么在tutorials中，让你复制一个**build.xml**文档，然后用**Ant**运行，其实在这个**build.xml**中，定义了一个Ant Task,它会下载ivy．也就是说，如果你已经运行了这个脚本，那么就不需要再去官网下载包并解压拷贝了．但是建议还是去下载，因为官网的包中，包含了大量例子和文档．

装好了**Ant+ivy**之后，就可以简单的通过一条**ant**命令进行编译了．

对于熟悉使用Maven的朋友来说，可能会觉得有点陌生．我之前也不了解这个，甚至没有听说过ivy,但是现在确实觉得是我经历过的最简单的编译过程．只不过其编译脚本比较复杂繁琐，如果是我们开发人员写的话，有点麻烦．但是其逻辑其实也不复杂．
