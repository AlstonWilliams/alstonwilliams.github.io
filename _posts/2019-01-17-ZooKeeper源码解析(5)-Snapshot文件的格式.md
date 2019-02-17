---
layout: post
title: ZooKeeper源码解析(5)-Snapshot文件的格式
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
在ZooKeeper的Snapshot文件中，存储了当时ZooKeeper的状态和数据．

那么ZooKeeper中到底存储了什么内容呢？官方文档中没有明确说明，所以我们只好通过读源码来获取．

我是在读完相关的部分的源码之后，总结出来的这样一个结构．我是通过源码中相关类的**serialize()方法和deserialize()方法**来总结的．


![](http://upload-images.jianshu.io/upload_images/4108852-0b1eeae4b64ed47f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面我们将会详细解释其中每一部分的作用．

## File Header

其实每种文件的十六进制的表示中，开头的部分都是Header，ZooKeeper的Snapshot的Header的结构如下:


![](http://upload-images.jianshu.io/upload_images/4108852-c31bddc262c6b2ad.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，在每个格子中，上面都是一个名称，下面是这个字段所占用的字节数．后面的图片中，我们都将采用这种方式来表示字段．

其中第一部分很好理解，就是一个固定的字符串－**＂ZKSN＂**，只有当Snapshot的magic字段是这个字符串以及以特定的方式结尾，那么这个Snapshot才是有效的．

第二个字段是**version**，它代表的是版本号．

第三个字段是**dbid**．那么什么是**db**呢？在ZooKeeper中，抽象出来一种数据类型－**＂ZkDatabase＂**，它在内存中维护着ZooKeeper Server的各种状态，比如**sessions，datatree以及committed logs**．**sessions**是ZooKeeper Server和Client中维护的一个连接，**datatree**是ZNode的形成的一棵树，这个很容易理解嘛．**committed logs**指的是已经提交的日志，这个是Zab中的一个概念．

## Other server's information

![](http://upload-images.jianshu.io/upload_images/4108852-0c4f2c1107a6b284.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**count**字段指的是更有多个ZooKeeper Server．

接下来是**count**个**server information**了，每个**server information**都包括**id**和**timeout**两部分．

**id**是ZooKeeper Server的id，ZooKeeper在Cluster模式下启动时，我们需要在**datadir**下面创建一个名为**myid**的文件，其中的数字就是ZooKeeper Server的id．

**timeout**是这台ZooKeeper Server和id为**id**的那台ZooKeeper Server之间的超时的时间，当超过了这个时间，就认为那台ZooKeeper Server不可用了．

## ACL Caches
![](http://upload-images.jianshu.io/upload_images/4108852-2999b26afb2c193d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一个字段是**map**，其实ACL Caches的结构就是一个**Map<Long, List<ACL>>**．这个Map中，key是一个分组的id，value是ACL列表．**map**字段表示的是Map的大小．

后面的就很容易解释了，**long**表示的是分组的id，**acls**表示的是这个分组内更有多少个ACL．**perms**是表示的权限，在ZooKeeper中，提供了下面的几种权限：


![](http://upload-images.jianshu.io/upload_images/4108852-5300184718f61157.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**scheme前的len**表示的是**scheme**的长度，因为**scheme**是String类型的，我们需要用它来指明它的长度．**id前面的len**同样也是表明**id**的长度表明的是**id**的长度，同样也是因为**id**是String类型的．

ZooKeeper都支持下面几种Scheme：

![](http://upload-images.jianshu.io/upload_images/4108852-77b474d7d2c590c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Nodes


![](http://upload-images.jianshu.io/upload_images/4108852-59f3bc00cce8df44.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Node指的是ZooKeeper中的ZNode．每个Znode都由几部分组成：路径，数据，元数据．每个字段的含义在ZooKeeper文档中有，这里我直接贴出来：


![](http://upload-images.jianshu.io/upload_images/4108852-71ab4dd5367af6e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根目录**＂/＂**有特殊的形式，它只存储**"1/"**，表示其长度为1，路径是**"/"**，在ZooKeeper的Snapshot中，就是用**header中的magic**以及**这个根目录**来判断文件是否是Snapshot的．
