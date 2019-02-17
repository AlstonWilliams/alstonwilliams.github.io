---
layout: post
title: 解决docker-pull时Connection-Reset的问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Docker
tags:
- Docker
---
由于众所周知的原因,当我们从docker.io pull镜像时,很可能会出现 Connection Reset或者Read time-out这类错误.

我们可以使用daocloud的加速器服务,来解决这个问题.详情请自行去daocloud上查看加速器一节.
