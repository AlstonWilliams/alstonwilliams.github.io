---
layout: post
title: ShadowSocks-libev从源码安装需要的包
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
先安装下面的两个包:
**apt-get install libpcre3 libpcre3-dev**
上面那个包时安装pcre的

然后在安装asciidoc:

在*/etc/apt/source.list*中加入下面这行:

**deb http://ftp.de.debian.org/debian jessie main**

然后使用
**apt-get update
apt-get install asciidoc**
安装一下就可以

然后就可以编译ShadowSocks-libev这个包了．如果不安装这两个包，在编译时会提示错误
