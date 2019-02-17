---
layout: post
title: YARN开启Debug模式
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在研究YARN的源码时，由于`info`模式下，日志数量实在是有限，不能更加全面的了解。于是，就寻找了一下YARN中启动DEBUG模式的方法。

经过搜索，可以在**yarn-env.sh**这个文件中，添加这么一行就好了：
~~~
export YARN_ROOT_LOGGER="DEBUG,console"
~~~

这样就可以开启Debug模式，查看更多日志了
