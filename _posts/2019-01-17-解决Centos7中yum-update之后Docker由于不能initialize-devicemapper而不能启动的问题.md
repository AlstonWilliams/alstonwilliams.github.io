---
layout: post
title: 解决Centos7中yum-update之后Docker由于不能initialize-devicemapper而不能启动的问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 容器
tags:
- 容器
---
在Centos7中，做Kubernetes实验，中间由于输出的结果跟我看的书上不一致，打算升级一下看看．

使用**yum update**升级之后，就出现了如题所示的错误．

找了半天，好不容易找到了答案．这里摘录下来:

>I found this problem after a yum update in Red Hat. Delete (or move) devicemapper folder and start docker, it will be recreated and docker will start normally:
$ rm -rf /var/lib/docker/devicemapper
$ systemctl start docker

这样虽然能启动起来Docker了，但是我们启动那些已经退出的容器，却是不能成功启动的，会提示没有device.所以我们还是直接删掉*/var/lib/docker*目录，然后直接重装就好．
