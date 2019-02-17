---
layout: post
title: 单节点Kubernetes集群配置总结
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Kubernetes
tags:
- Kubernetes
---
想尝试一下Kubernetes，但是之前都因为无法配置而放弃．有如下三种配置方式:

- 通过kubeadm配置．官方文档中就是用的这种方法．
- 通过自己下载Kubernetes的对应的包，自己配置来实现．
- 在Centos中通过从仓库中安装来实现．

开始我是尝试第一种方案．但是无奈因为大中华的网络原因，放弃了．

第二种方案，配置相当繁琐．尝试到一半，因为发现了第三种方案．就放弃使用第二种了．

至于第三种方案，是最简单．这也是为什么大多数文章中，都使用这种方案．因为我们可以直接从仓库中安装，没有大中华特有的网络问题．特别方便．目前在ubuntu中，仓库中似乎并没有kubernetes．所以我们使用centos7．

我们使用Vagrant在VirtualBox中新建一个Centos7的虚拟机.**Vagrantfile**文件的内容如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-95dc8620eeadec3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们设置其网络为*public_network*,这会让这个虚拟机和主机在同一个子网中．这里我的子网是**192.168.0.0/24**.

然后使用**vagrant up**命令来启动这个虚拟机即可．初次启动应该会挺慢，因为会下载box,进行配置等．

启动完成后，我们就可以通过**vagrant ssh**命令，进入虚拟机内部，并执行后面的命令了．

当然，你也可以自行下载一个centos7的镜像，然后在虚拟机里配置，但是这种方式比较繁琐，不推荐这样做．

具体的操作过程，请自行查看<KUBERNETES权威指南:从DOCKER到KUBERNETES实践全接触>这本书中的"从一个不简单的Hello World例子说起"这节．照着书中说的，一步步做即可．
