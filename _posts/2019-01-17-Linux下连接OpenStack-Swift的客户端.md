---
layout: post
title: Linux下连接OpenStack-Swift的客户端
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 小工具
tags:
- 小工具
---
昨天要验证一个配置文件是否正确，然而以前只是通过accesskey，secretkey这些，通过s3cmd来连接。这次需要通过username和password来连接。

在Windows下很简单，有一个叫做**CloudBerry Explorer**的软件，但是对于不是Windows而用Linux的我，就十分心急了。

经过搜索，发现Python有一个小工具，叫做python-swfitclient，可以解决这个问题。

安装非常简单，只需要通过下面的命令就可以安装好了:
**sudo pip install python-swiftclient**

然后，通过下面的命令就可以连接了：
**swift [-A *Auth URL*] [-U *username*] [-K *password*] stat**

其中最后的`stat`指的是你要执行的命令。

例如，我们可以这样来连接到**OpenStack Swift**:
**swift -A http://production01.acme.com/auth/v1.0 -U user01 -K password stat**

如果想了解更多的信息，请查看这个工具的官网[https://www.swiftstack.com/docs/integration/python-swiftclient.html](https://www.swiftstack.com/docs/integration/python-swiftclient.html)
