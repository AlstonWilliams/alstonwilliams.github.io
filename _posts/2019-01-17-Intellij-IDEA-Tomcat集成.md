---
layout: post
title: Intellij-IDEA-Tomcat集成
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
最近在做一个Web项目，由于Linux上的Eclipse丑的不能直视，操作又各种不方便，终于受够了Eclipse，重新用起了Intellij IDEA．

其实好久之前，就想过放弃Eclipse，用Intellij IDEA进行开发，但是由于Intellij IDEA不支持Tomcat自动重新部署，就没有转到Intellij IDEA，而是一直用Eclipse．

今天，又捣鼓了一番，虽然还是无法实现重新部署，但是也进步了一大步了．

在这里，我会一步步地说明如何配置Intellij IDEA，让它能够正常的使用Tomcat．

## 环境

我用的是Intellij IDEA Ultimate 2016.3版本，不知道Intellij IDEA Community Edition支不支持．

## 步骤

首先，创建一个Web项目，可以创建一个maven的web项目，也可以创建一个普通的dynamically project．

如果你跟我一样，是将项目从Eclipse迁移到Intellij IDEA，那么你还需要做一些额外的操作，比如，移除不存在的依赖，或者添加依赖等．

然后，我们需要确认Intellij IDEA中是否安装了**Tomcat and TomEE Integration**插件：


![](http://upload-images.jianshu.io/upload_images/4108852-ff6674c119573bb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

确保这个插件已经被安装并且开启之后，我们就需要对项目进行进一步的配置．

点击顶部菜单栏中的**Run -> Edit Configurations..**，在进入的面板中，点击左上角的绿色的**+**，然后在弹出的面板中，选择**Tomcat Server -> Local**：

![](http://upload-images.jianshu.io/upload_images/4108852-aba69fe1d04d87d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

创建完之后，就会弹出上图中右边的那个部分．


![](http://upload-images.jianshu.io/upload_images/4108852-b989eee211f34bf6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你的界面很可能跟我的不一样．

第一个需要注意的地方，是**JMX port**．你的应该是**1099**．这里要根据你的Tomcat的设置进行配置．因为Intellij IDEA会使用JMX进行项目的部署，监控等．

由于在我的机器上，Tomcat的**setenv.sh**中，已经开启了JMX，并且其端口号是**9999**，所以我把**JMX port**改成了**9999**．


![](http://upload-images.jianshu.io/upload_images/4108852-d152c0d6e092c18c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你的机器上Tomcat的**setenv.sh**中并没有配置JMX，那么你可以跟上图中一样，手动开启JMX并设置一下**JMX port**．

否则的话，会由于无法通过JMX连接到Tomcat，而导致项目部署不上去．

另一个需要注意的地方就是**On 'Update' action**这里，这一项表示当我们执行**Update**时，进行什么操作，这里我选择的是重新部署项目．因为我尝试过**Reload Classes and Resources**，但是并没发现有什么用．

那么什么是**Update**呢？当我们点击一次**Run -> Update Tomcat8 Application**时，就会触发一次**Update**．

需要注意的是，只有当项目启动后，这一项才可以点击．

设置好之后，我们就需要告诉Tomcat Server我们需要部署哪一个项目了．

点击**Server** tab旁边的**Deployment** tab，如下图鼠标处所示：

![](http://upload-images.jianshu.io/upload_images/4108852-7e153ab3bad6cfdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

点击完之后，会看到如上图所示的tab．

点击右侧的绿色的**+**，会让你选择要部署的Artifact．这里由于我们只有一个，所以它会直接选择．添加好之后，配置好右侧的**Application Context**．

![](http://upload-images.jianshu.io/upload_images/4108852-b9a91bf31c79b2b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么什么是**Artifact**？

如果在上面你选择**Artifact**时，并没有发现．那么你可以点击菜单栏中的**File -> Project Structure**，在其中我们就能发现**Artifact**一栏：

![](http://upload-images.jianshu.io/upload_images/4108852-e38aac00cd62c025.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后点击左上角的绿色的**+**号，按照下面的操作进行下去，就能创建一个**Artifact**：


![](http://upload-images.jianshu.io/upload_images/4108852-0618934caa254132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后回到前面的那个窗口中，将这个Artifact部署上去．

上面的操作都完成后，我们在底部会发现我们创建的Server．


![](http://upload-images.jianshu.io/upload_images/4108852-a31ceed4a5281814.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
