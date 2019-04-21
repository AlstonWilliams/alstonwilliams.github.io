---
layout: post
title: Ubuntu连接到ftp服务器
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
在实训时，由于要求使用Windows系统，而我使用的是Ubuntu系统．了解到Windows上的连接到Server时，使用的是ftp协议，所以我们完全可以自行连接．只不过麻烦一点．

那么到底如何来做呢？

首先，安装ncftp包．

安装好之后，便可以通过**ncftp**来进行ftp.

进入之后，会进入如下界面:


![](http://upload-images.jianshu.io/upload_images/4108852-362c526b9f93b055.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们就需要连接到ftp服务器.通过**open -u user_name host**来进行连接:

![](http://upload-images.jianshu.io/upload_images/4108852-8771e3ecae9a321f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后，使用**ls**命令就能看到全部的文件．
