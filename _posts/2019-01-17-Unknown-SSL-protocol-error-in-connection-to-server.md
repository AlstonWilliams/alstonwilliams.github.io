---
layout: post
title: Unknown-SSL-protocol-error-in-connection-to-server
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 跨越GFW
tags:
- 跨越GFW
---
上面的这个错误，相信在你**Curl**一个被GFW屏蔽的网站时，经常会遇到．这篇文章中，会介绍在中国发生这个错误最可能的原因．

在我打算搭建一个Kubernetes集群时，打算使用http_proxy来访问gcr.io这些网站．

下面说一下我的步骤:

- 安装ShadowSocks命令行客户端
- 设置http/https代理:**export http_proxy=socks5://127.0.0.1:1080 
 export https_proxy=socks://127.0.0.1:1080**
- 先使用下面的方法测试:**curl -v https://www.baidu.com**．发现一切正常．
- 然后使用google测试: **curl -v https://www.google.com**．这时候，就遇到上面的那个错误了．

其实，之前在介绍搭建HTTP/HTTPS代理服务器时，我也遇到了这个问题．当时网上查了很多东西，都不知所云．也就一直没有解决．而今天，我找到了答案！！！

出现那个错误的原因，就是各大运营商中的DNS服务器中，将这些GFW屏蔽的网站，设置了一个错误的Ip.

我们实际看一下:

我们先看我在本机上，查看到的google的Ip:
![1.png](http://upload-images.jianshu.io/upload_images/4108852-987324a64564bd8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，*www.google.com*这个域名，它给的ip都是*93.46.8.89*,结尾还说没有发现主机名．

然后，我们来看在国外的vps上，看到的google的Ip:
![2.png](http://upload-images.jianshu.io/upload_images/4108852-19cfdbd589d2a853.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，国外的vps解析出来的Ip跟国内的完全不同．

证据就在这里!!!

那我们如何来解决这个问题呢?

用过Linux的朋友已经得到答案了．无非就是在hosts文件中添加上一条记录呗:
**127.217.5.206 www.google.com**

然后，我们就可以正常的访问Google了:

![3.png](http://upload-images.jianshu.io/upload_images/4108852-d7dacff6beabf9e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这种方法的优点是简单，只需要添加需要的域名的记录即可．缺点是，无法共享这些记录．

而我们在一个集群中，有好多机器，不可能一个个的修改hosts文件．当然，使用Chef这些自动化部署工具也能简化一定的工作量．

那更好的方法是什么呢?

搭建一个DNS服务器．然后让我们的集群中的服务器使用这个DNS服务器．只是这个现在我也还没尝试过，这两天尝试搭一个．成功后，会把过程分享给大家．

那为什么用浏览器就不需要设置hosts文件，直接就能访问Google呢？因为浏览器中会使用Google的DNS服务器．

当时搭建HTTP/HTTPS服务器时，也是这个问题．这篇文章中的方法，就是解药．
