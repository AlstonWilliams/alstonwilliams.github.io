---
layout: post
title: Java-SocketException-Too-many-open-files
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
在通过ab进行压力测试时，遇到了这个问题．并发请求数为10000.

因为每个Socket都对应一个文件，所以同时这么高的并发请求势必会导致创建很多文件．而默认情况下，Linux中，用户最多可创建的文件数为1024.可以通过**ulimit -n**来查看．

了解其原因之后，我们就能想到解决方案了．无非就是增加用户可创建的文件数．通过编辑**/etc/security/limits.conf**文件，添加**user_name - nofile max_open_file**一行:


![](http://upload-images.jianshu.io/upload_images/4108852-2edbdc3d052d34d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后重启一下机器，就可以看到用户最大可打开文件数为20000了．

这时再并发请求10000就不会有问题了．
