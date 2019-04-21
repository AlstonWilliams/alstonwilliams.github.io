---
layout: post
title: Centos　Kubernetes集群如何添加一个新的节点
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 容器
tags:
- 容器
---
在生产环境中，我们不可避免的会涉及到节点的动态添加与删除．下面我们就来介绍一下如何添加一个新的节点．

实验环境，两台Centos7,均通过vagrant创建．其中一台作为master与node节点，另一台作为node节点.Vagrantfile文件的内容均为下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-b96ec5633076b7bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们先来介绍一下如何创建那个集master和node节点于一点的机器．

先安装docker,etcd,kubernetes,通过下面的这条命令:
**sudo yum install -y docker etcd kubernetes**

禁用Centos的防火墙:
**sudo systemctl disable firewalld
sudo systemctl stop firewalld**

配置一下Docker.在*/etc/sysconfig/docker*文件中，设置其中的OPTIONS内容如下:


![](http://upload-images.jianshu.io/upload_images/4108852-37eb8462fbdef3f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


其中*--insecure-registry*这部分自行设置自己的就好了，我这里设置的是我的仓库以及google cloud container的仓库地址．

我们还需要配置docker使其让当前用户无需sudo即可使用:
** sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo systemctl restart docker
sudo reboot now**

然后，配置Kubernetes Apiserver,配置文件为*/etc/kubernetes/apiserver*,使其可以被远程访问到，默认情况下，应该是只有本地能访问.同时，去掉*--admission_control*中的**ServiceCount**:


![](http://upload-images.jianshu.io/upload_images/4108852-3369127d27f670ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后，按顺序启动所有服务就好啦:
**sudo systemctl start etcd
sudo systemctl start docker
sudo systemctl start kube-apiserver
sudo systemctl start kube-controller-manager
sudo systemctl start kube-scheduler
sudo systemctl start kubelet
sudo systemctl start kube-proxy**

此时，通过**kubectl get nodes**，我们能看到如下结果:


![](http://upload-images.jianshu.io/upload_images/4108852-c01f11b55ea6dadf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后，我们设置另一台Centos为另一个node，并使其连接到kubernetes上集群上．

还是通过**sudo yum install docker kubernetes**安装必要的包．因为它只是一个node节点，所以*etcd*这个服务不需要．实际上，*kubernetes*这个包中的很多跟master节点相关的服务也是不需要的．

然后，同样修改docker的配置文件，还是如上面的图片中那样配置，这里我就不再介绍了．

然后，因为我们需要让*kubelet和kube-proxy*能够自动连接到master节点上，所以我们还需要配置*kubelet*和*kube-proxy*使用master节点上的apiserver的地址和端口.

我们先配置kubelet，配置文件为*/etc/kubernetes/kubelet*，进行如下图所示的配置．这里我们的master节点上的apiserver的地址为**192.168.1.107:8080**.同时，我还配置了一下这个kubernetes node的主机名:


![](http://upload-images.jianshu.io/upload_images/4108852-9db3763b733f73fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来，我们配置*kube-proxy*,配置文件为*/etc/kubernetes/proxy*:


![](http://upload-images.jianshu.io/upload_images/4108852-0aef417765e5bf2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这样就配置好了．

然后，我们启动需要的服务即可:
**sudo systemctl start docker
sudo systemctl start kubelet
sudo systemctl start kube-proxy**

然后，在master节点上，我们就能看到这个节点加入成功了:

![](http://upload-images.jianshu.io/upload_images/4108852-4dfec451dc78edb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


好，成功之后，我们就可以进行更多的操作了．
