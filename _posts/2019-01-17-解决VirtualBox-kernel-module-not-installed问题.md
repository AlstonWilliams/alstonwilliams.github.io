---
layout: post
title: 解决VirtualBox-kernel-module-not-installed问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
最近在用Vagrant配合VirtualBox来做Kubernetes的实验时，一直都遇到如题所示的错误。花了好长时间都没解决，甚至都重装系统了，还是没解决。

今天深入的搜索了一下答案，搜索一下编译VirtualBox kernel module时遇到的一个错误:modprobe vboxdrv failed.竟然搜到了正确答案。

原因是，因为机器上开启了Security Boot,所以机器不会加载那些它不信任的模块，就导致了如题所示的错误。

解决的方案是，关闭Securiry Boot的这个选项。在启动时，根据你的机器型号，按不同的键进入BIOS，然后关闭它就可以了。
