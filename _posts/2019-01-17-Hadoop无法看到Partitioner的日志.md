---
layout: post
title: Hadoop无法看到Partitioner的日志
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
正常情况下，我们在自定义的**Partitioner**中输出的日志，会在Mapper的日志中看到。

但是，有一种情况下，看不到。就是我们的**Partitioner**根本没有被调用的情况下，看不到。

你可能会想，这还用你说？

但是，有的时候，我们明明定义了**Partitioner**，它却没被调用。

那么这是什么时候呢？

当我们的**Reducer**只有一个时，**Partitioner**根本就不会被调用，所以当然看不到输出。这点谨记！
