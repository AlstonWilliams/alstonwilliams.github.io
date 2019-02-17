---
layout: post
title: 解决Centos7中localhost不起作用
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
之前就有这个问题，但是一直没有什么太大的问题，就没解决．然而在配置Kubernetes dashboard时，因为默认的**apiserver**的hostname就是localhost,导致不能正常启动．这才想解决．

很奇怪，**ping localhost**时，解析出来的ip就是**127.0.0.1**.但是使用curl等命令时，就不能正确解析了．

最后，通过删除**/etc/hosts**文件中的重复的*localhost*项，解决之．
