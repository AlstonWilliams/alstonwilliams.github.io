---
layout: post
title: HBase本地安装Snappy
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
笔者最近要调试一个线上的HBase bug，所以需要做到本地配置等跟线上完全一样。其它的都还好说，但是到了Snappy这儿却碰了一鼻子灰。

所以，在这篇文章，我会介绍如何在本地，安装Snappy，并配置HBase使用它。

## 环境

HBase cdh5-1.2.0_5.12.1

环境特别重要。它直接关系到需要使用的Snappy以及Hadoop的版本。如果版本对不上，很可能出现链接本地动态链接库时的错误。

## 操作

操作其实很简单。但是由于官网上没有写，我们自己找blog又没有明确说明版本的问题。所以会碰不少灰。

1. 编译或者从生产环境，将对应的libsnappy.so拷到本地。
这一步，你可以选择自行编译snappy，但是，最好不要自行编译，因为你需要非常清楚地知道，你依赖的snappy是哪个版本。这个是[1.1.3版本](https://github.com/google/snappy/releases/download/1.1.3/snappy-1.1.3.tar.gz)的。
最好还是直接从生产环境拷过来，因为你本地的HBase源码，肯定对应的是生产环境的。如果使用的是CDH HBase，路径应该是`/opt/cloudera/parcels/CDH/lib/hadoop/lib/native `

2. 编译或者从生产环境，将对应的libhadoop.so拷到本地。
同样，非常不建议自行编译libhadoop。因为我看源码，它依赖的是cdh自己的版本，而且，编译的过程中很容易碰到错误。也不建议直接从网上找一个libhadoop.so。
还是直接从生产环境拷过来就好了。路径同上

3. 在HBASE_HOME下面创建一个目录`lib/native/Linux-amd64-64/`，或者是`lib/native/Linux-i386-32 /`，取决于你的CPU架构。

4. 将上面的两个so文件拷贝到这个目录里面去

5. 在hbase-env.sh中，添加这么两行:
~~~
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HBASE_HOME/lib/native/Linux-amd64-64                                                                                              
export HBASE_LIBRARY_PATH=$HBASE_HOME/lib/native/Linux-amd64-64
~~~

6. 这就完成了，重启即可。

重点是，上面两个动态链接库必须有。而且版本必须对应。这一点要谨记。
