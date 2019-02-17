---
layout: post
title: 解决automake不生成Makefile-in的问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- C语言
tags:
- C语言
---
我用的automake的版本是1.14.1,按照一个tutorial来做的时候,一直不能用automake产生Makefile.in文件.错误提示如下:

**Makefile.am: error: required file ./AUTHORS not found**

完全是照着tutorial上来做的,为啥它的能行,我的就不行呢?

找了大量的文档,也没有知道答案.

最后,我创建了缺少的文件.比如,上面提到的**./AUTHORS**文件.然后,再重新执行automake命令,他竟然就行了...

那个tutorial中,使用的是**automake --add-missing**这条命令,这条命令应该会自动给我们创建缺少的文件.然而,我运行时,却并没有创建,于是,就那样了.

可是我以为它会创建,也认为这些文件无关紧要,就没往深处想.谁知竟然是因为缺少这些文件,导致automake不能执行成功!!!

不知道这个问题是不是automake1.14.1中的一个bug.就这样吧.
