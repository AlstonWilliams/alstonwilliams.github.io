---
layout: post
title: Ubuntu14-04-安装Watchman
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
打算重写项目的UI.因为之前用的国内的某家公司的html5来做的APP的前台,而这家公司的这个产品可能还不成熟,而导致做出来的APP的前台老是有一些问题.于是就打算采用React Native来进行重写.

第一步是搭建React Native的开发环境.这一步就花费了我六个小时左右的时间.主要就是卡在Watchman的安装上.

Watchman是Facebook出品的一个用于监控文件变化的小程序,它在React Native的Hot reloading中起着非常重要的作用.我们也知道,要是没有Hot reloading,我们开发的效率会大大降低.所以不管怎样,都得装上这款工具.

可问题是,照着官网的步骤一步步来做,到最后还是没安装成功.老是提示**libpcre.so.1**这个共享库找不到.

官网上的步骤如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-74a28f0f5aab58d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当时也没细看这段代码上面的描述.就照着执行而已.

现在在截上面那个图时,注意到官网的描述.既然缺少**libpcre.so.1**这个共享库,那我们完全可以启用**python**支持,而不启用**pcre**支持.

![](http://upload-images.jianshu.io/upload_images/4108852-839678aab445f43c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


那我在不知道可以这么解决之前,是如何解决的呢?

安装**homebrew**工具,用下面这条命令:
**ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install)"**

然后修改环境变量:

**echo 'export PATH="/home/alstonwilliams/.linuxbrew/bin:$PATH"' >>~/.bashrc
echo 'export INFOPATH="/home/alstonwilliams/.linuxbrew/share/info:$INFOPATH"' >>~/.bashrc**

让修改生效:

**source ~/.bashrc**

用**homebrew**安装python:

**brew install python**

最后,安装**watchman**:

**brew install watchman**

我们可以通过下面的命令来验证是否安装成功:
**watchman -v**

输出版本号则表示安装成功.

其实完全不用这么麻烦,就在编译安装时,不用pcre,而用Python就行.
