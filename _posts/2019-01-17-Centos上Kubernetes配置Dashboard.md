---
layout: post
title: Centos上Kubernetes配置Dashboard
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Kubernetes
tags:
- Kubernetes
---
如果使用官方的tutorial来安装配置Kubernetes集群，Dashboard这个模块应该已经自带了．然而，谁让我们大中华享受不到那种待遇呢?

这里我们还是通过**https://github.com/kubernetes/dashboard**这上面的那个脚本来启动Dashboard．但是，因为在Centos7上从仓库安装的Kubernetes会有一些默认的配置，这些配置使我们不能正常的只是用那个脚本来创建一下相应的Service就好．所以，我们还需要修改一些配置．如果你不修改的话，就会因为无法访问ApiServer而启动不了Dashboard.

首先，修改*/etc/kubernetes/apiserver*文件，在修改之前，内容应该如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-9ea5ae227d453685.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们修改其中的**KUBE_API_ADDRESS **那部分，将其修改成任何地址都可以访问:


![](http://upload-images.jianshu.io/upload_images/4108852-ffcc1f7bd2705c59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


默认情况下，其配置是只能本地访问，而因为Dashboard运行在由Docker daemon划分的一个子网的一个容器中，相当于和Kubernetes Apiserver并不是在同一个网络中，所以访问不了．将ApiServer配置成从任何地址都可以访问，就可以解决这个问题．应该也有其他的方法也可以解决此问题．但是这样最为简单．

然后，通过下面的命令重启ApiServer:
**sudo systemctl restart kube-apiserver**

然后，我们配置Dashboard连接的ApiServer的地址．在我们上面下载的那个脚本中的第54-58行，有关于它的配置，但是默认是被注释掉的:


![](http://upload-images.jianshu.io/upload_images/4108852-1f622a4335677a28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们需要将其换成** http://docker_bridge_address:apiserver_port**．其中*docker_bridge_address*是docker daemon创建的网桥的地址，默认是** 172.17.0.1**.也不一定，你可以在主机上通过** ifconfig 或者 ip addr**命令查看．*apiserver_port*是Apiserver的端口号，如果你没有设置过，应该是** 8080**:


![](http://upload-images.jianshu.io/upload_images/4108852-1c3bbe10d6d9ac74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因为Docker容器是和docker daemon创建的那个网桥处于同一网段，而这个网桥的Ip实际上又是主机对Docker容器而言的Ip地址．所以这样Dashboard容器就可以访问主机上Apiserver接口了．

我们还需要配置Dashboard不使用*gcr.io*上面的镜像，而是使用*docker.io*上面的一个同步的镜像．就是如上图中的第49行中所示的那样．

然后，通过下面的命令来创建对应的Deployment等:
** kubectl create -f kubernetes-dashboard.yaml **

如果这条命令操作成功，我们会看到如下图所示的输出:

![](http://upload-images.jianshu.io/upload_images/4108852-b0f9e803e145be5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图中，我们可以看到，我们可以通过30090来访问这个Kubernetes-Dashboard.

我们查看一下* Deployment, Pods, Services*等:


![](http://upload-images.jianshu.io/upload_images/4108852-767b7d04e41573b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到，kubernetes-dashboard的Pod已经运行成功了．

最后，我们从浏览器中查看一下正确运行的结果:


![](http://upload-images.jianshu.io/upload_images/4108852-7074d8a02e3aff8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
