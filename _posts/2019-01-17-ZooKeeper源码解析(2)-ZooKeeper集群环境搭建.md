---
layout: post
title: ZooKeeper源码解析(2)-ZooKeeper集群环境搭建
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
为了学习一致性协议，研究一下分布式系统的实现，就选择了ZooKeeper这个分布式键值对系统．

其实好久之前就在学习一致性协议了，读完了Paxos,Raft以及Zab的论文，一直都觉得其实它们差不多，差别在于细节的实现上，所以分别找了Paxos, Raft的开源实现打算研究，由于Zab是集成在ZooKeeper内部的，没有单独拿出来的实现(也搜到过，但是不敢保证其正确性)，就看ZooKeeper相关的源码来理解．

我感觉ZooKeeper的核心就是Zab，拆分一下模块的话，还有几个小的模块，比如验证的模块，监控的模块，但是这些倒是都不难实现．

之前的一段时间，一直在学习别的东西，没有时间来阅读ZooKeeper的源码，最近正好有时间，就深入研究一下Zab的实现．

也正是因为最近开始研究了，而ZooKeeper默认情况下，配置使用的是开发环境，即只有一个实例．这样对于我们研究Zab算法肯定是不够的，只有一个实例还需要什么一致性算法，还需要什么Zab．所以，我们要做的第一步就是搭建一个ZooKeeper集群环境．

## 工具

这里先说一下需要的工具，以及我运行的平台．

- 机器: Ubuntu 14.04，内核版本是3.19.0-25-generic
- 虚拟机软件: Virtual Box 5.1.12
- 虚拟机操控软件: Vagrant 1.9.1
- 虚拟机box: ubuntu/trusty64
- ZooKeeper 3.4.9版本的源码

其实这里本来打算使用Docker的，毕竟它相对于虚拟机来说更加轻量级，但是由于当时考虑上有点偏差，最终就采用了Vagrant+Virtual Box，各位在搭建时，完全可以采用Docker的．

## 搭建过程

首先，编写Vagrantfile，其内容如下：

~~~~
servers=[
  {
    :hostname => "zookeeper1",
    :ip => "192.168.56.101",
    :box => "ubuntu/trusty64",
    :ram => 512
  },
  {
    :hostname => "zookeeper2",
    :ip => "192.168.56.102",
    :box => "ubuntu/trusty64",
    :ram => 512
  },
  {
    :hostname => "zookeeper3",
    :ip => "192.168.56.103",
    :box => "ubuntu/trusty64",
    :ram => 512
  }
]

Vagrant.configure("2") do |config|
  servers.each do |server|
    config.vm.define server[:hostname] do |node|
      node.vm.box = server[:box]
      node.vm.hostname = server[:hostname]

      node.vm.network :private_network, ip: server[:ip]

      node.vm.synced_folder "/home/alstonwilliams/source_code/zookeeper-3.4.9", "/home/vagrant/zookeeper"

      node.vm.provider :virtualbox do |v|
  	  v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
	  v.customize ["modifyvm", :id, "--memory", server[:ram]]
      end
    end
  end
end

~~~~

在上面的Vagrantfile中，我们创建了三台**ubuntu/trusty64**的虚拟机，为其设置了主机名，IP地址，内存等，并且设置了一下文件共享．因为我们需要在本机上修改ZooKeeper，然后在本机上编译之后，将其同步到虚拟机中，然后分别在三台虚拟机中启动它，组成一个集群．当然，你也可以不用通过这种方式同步，你可以通过**sync**命令或者**scp**命令，每次编译完后，分别手动将其发送给三台虚拟机，但是这样毕竟增加了操作成本嘛．

然后通过**vagrant up**命令创建并启动这三台虚拟机．需要注意的是，如果你的本机上并没有**ubuntu/trusty64**这个box，那么在下载这个box的时候，很可能会由于GFW而造成下载失败．解决方案很多，比如使用代理啊，或者用海外的服务器下载下来，然后再在海外的服务器上搭设一个FTP服务器，然后主机在用FTP的方式从海外的服务器上下载下来并导入本机啊，等等．

创建好之后，由于ZooKeeper的运行还需要Java，所以我们在三台虚拟机中需要分别运行下面的命令安装OpenJDK：

**sudo apt-get update 
sudo apt-get install openjdk-7-jdk**

这样的话，运行ZooKeeper集群的硬性资源我们就都有了．

为了在运行ZooKeeper输出更加详细的调试信息，我们需要修改一下ZooKeeper的日志配置文件，然后手动编译一遍．当然，如果你不想调试，只想搭建一个集群，然后让它能够正常运行的话，那么这一步是不需要的．

在ZooKeeper的源码的**conf**目录中，我们可以看到有一个**log4j.properties**文件，原本在这个文件的最上面的两行是:

~~~~
zookeeper.root.logger = INFO, CONSOLE
zookeeper.console.threshold = INFO
~~~~

在这种配置下，只有**INFO**及以上等级的日志才会输出到控制台中，所以**DEBUG**等级的日志是不会输出的，我们将这两行改成下面这样，让其输出:

~~~~
zookeeper.root.logger = DEBUG, CONSOLE
zookeeper.console.threshold = DEBUG
~~~~


![](http://upload-images.jianshu.io/upload_images/4108852-8dcea3f506c45af9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后通过**ant**命令编译一下．

如果你已经正确配置好**Ant**以及**Ivy**,那么编译过程就是这么简单，而如果你还没有配的话，请参考我的这篇文章:[编译ZooKeeper](http://www.jianshu.com/p/dccdc43caf03)

编译好的jar文件位于ZooKeeper源码的**build**目录中．

其实上面的日志配置中，你也可以直接将**log4j.rootLogger=${zookeeper.root.logger}**修改成**log4j.rootLogger=DEBUG, CONSOLE, ROLLINGFILE**这样也行，但是由于现在我们的源码的目录是被三个虚拟机共享的，也就是三个ZooKeeper实例共享的，所以同时输出到同一个日志文件中，这些日志就难免有些不准确．所以我们还是采用第一种方式来输出DEBUG日志．

编译好ZooKeeper之后，我们还需要编写ZooKeeper集群配置文件，修改ZooKeeper源码目录下的**conf/zoo_sample.cfg**文件并保存为**zoo.cfg**，将其修改成如下图所示的内容:


![](http://upload-images.jianshu.io/upload_images/4108852-ee383960fdef13d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，其中修改的地方有，修改了**dataDir**的值，以及增加了集群中机器所在位置的信息．

之前**dataDir**写的是一个临时目录，这样虚拟机一关，这里面的内容就被清除了，而这个目录中存放着很多重要的内容，比如数据啊，事务日志啊，以及服务器的id，所以我们要将其修改为一个可以永久保存的位置．

机器所在的位置的信息采用的是**server.id=server_ip:2888:3888**的这种形式，这里由于我们有三台虚拟机，所以添加了三台服务器．后面的两个端口号一个是Leader的端口号，一个是进行选举时使用的端口号，这两个端口号都是固定的，至少我在ZooKeeper官网的配置选项中没有看到有哪个选项可以修改．如果这里可以修改的话，我们就不必大费周折启动三个虚拟机了．

然后，在三台虚拟机上，分别建立**/var/zookeeper**目录，并为其赋予正确的权限，同时，在三台虚拟机的**/var/zookeeper**目录中，新建一个名为**myid**的文件，分别对应其id．比如，在IP为**192.168.56.101**的这台虚拟机上，**myid**的内容为**1**，而在**192.168.56.102**的这台虚拟机上，**myid**的内容为**2**．

这个文件是必须的，服务器之间进行通讯时，就是用的这个id.

然后，在虚拟机中映射到的ZooKeeper源码目录下，分别执行下面的命令启动三个ZooKeeper实例:

**java -cp build/zookeeper-3.4.9.jar:lib/log4j-1.2.16.jar:lib/slf4j-log4j12-1.6.1.jar:lib/slf4j-api-1.6.1.jar:conf org.apache.zookeeper.server.quorum.QuorumPeerMain conf/zoo.cfg**

启动完成后，从三台虚拟机的控制台中，就能看到启动成功了，并且能看到一些Zab算法执行的过程:

![](http://upload-images.jianshu.io/upload_images/4108852-c938c478c5481936.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-fdc373b66194cd4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-d0079a35abc7a87a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样就大功告成了!就可以调试并研究Zab的实现了!
