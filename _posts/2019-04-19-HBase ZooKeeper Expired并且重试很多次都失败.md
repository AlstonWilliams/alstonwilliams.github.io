---
layout: post
title: HBase ZooKeeper Session Expired并且重试很多次都失败
date: 2019-04-19
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

最近在算数据的时候，碰到了这么一个问题。其实本身ZooKeeper Expired这个没什么问题，但是重试了很多次并且都没成功，就有问题了。

其实出现Session Expired这种问题的原因，无非就是几种:
- ZooKeeper挂掉了
- zookeepr连接数超出
- 超时时间过小

检查了一下，前几种情况，都是不存在的。那是什么原因呢？

后来偶然看了一下官网的一篇文章，提到如果client频繁的进行swap，即内存不够用，需要将内存中的数据频繁的swap到磁盘上，也会导致Session Expired这种问题。

回想我们的场景，就是需要将HBase中的Bitmap的数据进行放大，尽管我们分配了20GB的内存，但是明显不够用，后来从Spark中看到`Shuffle Spill(Memory)`有30TB+,`Shuffle Spill(Disk)`也有几十GB，于是猜测应该是这个问题导致的。

`Shuffle Spill(Memory)`和`Shuffle Spill(Disk)`的关系如下:
> Shuffle spill (memory) is the size of the deserialized form of the data in memory at the time when we spill it, whereas shuffle spill (disk) is the size of the serialized form of the data on disk after we spill it. This is why the latter tends to be much smaller than the former


遗憾的是，我们从Executor日志中并没有看到足够的信息来证明这一点，而且后来我们采取了另一种方案，也没有继续复现这个问题。不过本身场景已经能说明一切了。

相关文档如下:
- https://wiki.apache.org/hadoop/ZooKeeper/Troubleshooting
- http://www.openskill.cn/question/435
