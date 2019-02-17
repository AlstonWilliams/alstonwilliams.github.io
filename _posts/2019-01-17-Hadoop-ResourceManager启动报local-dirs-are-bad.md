---
layout: post
title: Hadoop-ResourceManager启动报local-dirs-are-bad
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
有的时候，我们通过**bin/yarn nodemanager**启动一个NodeManager时，我们在ResourceManager的输出中，能看到这么一个错误:**local-dirs are bad...**

那么这个错误是由于什么导致的呢？

最常见的原因，就是NodeManager所在的机器上，可用的磁盘空间超过了**max-disk-utilization-per-disk-percentage**这个选项设定的默认阈值90.0%

所以，解决方案有两个:
  - 第一个是，清理机器上的磁盘空间
  - 另一个是，调大这个阈值，比如：
**<property>
        <name>yarn.nodemanager.disk-health-checker.max-disk-utilization-per-disk-percentage</name>
        <value>98.5</value>
</property>**

还有一种方案就是禁止NodeManager进行健康检查，但是，这样有一个缺陷，就是当你的作业长时间卡顿在那儿，你不能通过查看日志来发现到底是哪里出现了问题。所以，这种方案最好不要使用。
