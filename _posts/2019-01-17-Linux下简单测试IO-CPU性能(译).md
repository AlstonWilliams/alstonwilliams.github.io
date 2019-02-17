---
layout: post
title: Linux下简单测试IO-CPU性能(译)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 小工具
tags:
- 小工具
---
原文地址:https://haydenjames.io/web-host-doesnt-want-read-benchmark-vps/

译者注：本篇文章只是非常简单的概括原文的内容，原文有更多的指导，所以建议直接阅读原文。

## 使用dd测试IO性能

1. 测试IO write性能。使用下面的命令即可:
~~~
dd if=/dev/zero of=diskbench bs=1M count=1024 conv=fdatasync
~~~
结果看起来应该是这样:
~~~
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 0.553284 s, 1.9 GB/s
~~~
2. 测试无buffer cache的读性能。使用下面的命令:
~~~
echo 3 | sudo tee /proc/sys/vm/drop_caches
dd if=diskbench of=/dev/null bs=1M count=1024
~~~
第一步的作用是，清空buffer cache。让第二步的读操作直接落到磁盘上。
输出结果如下:
~~~
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 0.425812 s, 2.5 GB/s
~~~
3. 测试有buffer cache的读性能。经过上面的操作以后，此文件已经存在与buffer cache中了。所以我们只需要重复上面的第二个操作就好了:
~~~
dd if=diskbench of=/dev/null bs=1M count=1024
~~~
输出结果如下:
~~~
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 0.135034 s, 8.0 GB/s
~~~
可以看到，速度提升了很多

## 使用dd测试cpu性能

只需要执行下面的命令:
~~~
dd if=/dev/zero bs=1M count=1024 | md5sum
~~~

结果如下:
~~~
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 2.5454 s, 422 MB/s
cd573cfaace07e7949bc0c46028904ff -
~~~
