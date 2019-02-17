---
layout: post
title: Intellij-IDEA-Golang插件让vendor-src目录生效
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
Docker 1.12.5版本的源码中，很多源码都是放在**vendor/src**目录下的，在Intellij IDEA中打开，就提示找不着路径．

在环境变量中，给GOPATH加上了vendor的路径，但是还是不生效．

最终，还是在Intellij IDEA中解决的．

解决方法如下:

打开**'Setting'**页面，并在其中找到**Language &Frameworks**中的**Go**,打开其下的**Go Libraries**.在右侧的**Project libraries**那里，加上**vendor**目录的路径．

![](http://upload-images.jianshu.io/upload_images/4108852-6ea767fed696c460.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
