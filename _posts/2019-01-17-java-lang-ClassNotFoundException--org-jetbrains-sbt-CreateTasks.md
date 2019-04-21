---
layout: post
title: java-lang-ClassNotFoundException--org-jetbrains-sbt-CreateTasks
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
在创建Scala项目时，报这个异常。

IDE: IntelliJ IDEA
SBT: 1.0.4
Scala: 2.12.4

SBT和Scala是IntelliJ IDEA自动安装的。

错误原因为，项目中使用了错误的SBT版本。

修改**project->build.properties**，将**sbt.version**修改成机器上的**1.0.4**，即可解决问题。
