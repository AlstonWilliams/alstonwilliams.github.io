---
layout: post
title: HBase强制disable表
date: 2020-04-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

笔者在使用HBase的过程中，遇到了一个非常蛋疼的问题。表死活disable不掉，页面上一直显示region在OPEN。

所以想把这张表禁用掉，然后重建一张。但是表有region在Open的时候，有个锁，所以disable也不现实。

Google到了一波骚操作，可以直接通过修改meta表的方式把表禁用掉：
~~~
put 'hbase:meta','insight:label_ds_merged_second','table:state',"\b\u0001"
~~~
这样禁用掉以后，重启一下集群就好了

详细资料可以参考这个：
https://community.cloudera.com/t5/Support-Questions/Hbase-table-is-stuck-in-quot-Disabling-quot-state-Neither/m-p/235112
