---
layout: post
title: 使用JMX来监控Tomcat内存使用情况
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
最近发现Tomcat中，出现了不能分配内存的情况．很纳闷为什么会出现这种情况．于是就打算分析一下Tomcat的内存使用情况．

## 前提
这里我使用的Tomcat是Tomcat8，因为它已经集成了JMX，我们只需要简单配置一下即可．其他的版本并没有尝试过．

## 配置
在Tomcat的根目录下的*/bin*目录中，创建**setenv.sh**文件，同时添加以下内容:

![1.png](http://upload-images.jianshu.io/upload_images/4108852-3c421a9ea67b914b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就这么简单的配置一下，再重启一下Tomcat就好．

## 使用
我们在命令行中输入**jconsole**,就可以连接到Tomcat8并进行简单的监控了:


![2.png](http://upload-images.jianshu.io/upload_images/4108852-f3ed720f5b3d1611.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![3.png](http://upload-images.jianshu.io/upload_images/4108852-5d718df364a86879.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
