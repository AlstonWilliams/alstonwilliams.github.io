---
layout: post
title: 解决Linux中cannot-open-shared-object-file-No-such-file-or-directory
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
在一个C项目中,我们需要解析配置文件,然后选用了libconfuse库.从源码编译安装之后,照着官网的例子,写了一个测试程序,却不能成功运行,老是出现如题所示的错误.

开始是找不到函数的定义,于是链接了一下外部库,解决:
**gcc -o TestConfuse TestConfuse.c /usr/local/lib/libconfuse.a**

然后就是cannot open shared object file: No such file or directory了,通过执行**sudo ldconfig**命令解决.

**ldconfig**是一个用于管理**Share Library Cache**的工具.这些缓存一般保存在**/etc/ld.so.cache**中.这些缓存被系统用于映射库名称和其位置的关系.这个映射关系,会在共享库需要被动态链接到程序中时,被用到.默认情况下,共享库都放在**/lib,/usr/lib**中.

那么,如果我们将共享库安装到**/usr/local/lib**中,在我们需要用到这个共享库时,就会因为从动态库缓存中,找不到它,而导致出现上面的错误.

解决方法有两个:
- 设置LD_LIBRARY_PATH这个变量,告诉系统,我们把共享库放到了这里.
- 重新构建**Share Library Cache**.通过**sudo ldconfig**即可进行重新构建.

在Ubuntu14.04上实验过没有问题,在其他的Linux发行版上没有尝试过.
