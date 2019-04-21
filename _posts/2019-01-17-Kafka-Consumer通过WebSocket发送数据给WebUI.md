---
layout: post
title: Kafka-Consumer通过WebSocket发送数据给WebUI
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
Kafka Stream API在处理完数据后，会将数据发送到我们预定义的topic．如果我们需要将这些数据发送给我们的WebUI，那么我们就需要写一个Consumer，让它订阅上面的那个topic,然后发送数据给WebUI.

那么到底如何来做呢？下面我会一步步的描述．

首先，我们需要新建一个WebApp．其目录结构大体如下:


![](http://upload-images.jianshu.io/upload_images/4108852-be0b3d6376335082.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们为什么需要新建一个WebApp项目呢?因为WebSocket是一个基于HTTP协议的协议，所以我们还需要使用Tomcat等服务器软件让其对外提供服务．



新建完这个项目之后，我们需要添加需要的依赖，依赖如下:


![](http://upload-images.jianshu.io/upload_images/4108852-a8c1b4b8e44632f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

依赖包括有WebSocket的依赖，Kafka的依赖．需要注意的是，**kafka_2.11**和**kafka-clients**版本号需要相同，否则可能会因为不匹配而造成意外的错误．我开始就是因为版本号不同而造成找不到类的错误．

另外，它们的版本最好跟你的Kafka的版本相同，我这里使用的是**kafka 0.11.0.0**，所以使用的也是这个版本的依赖．

我们就需要写WebSocket服务器端代码了:


![](http://upload-images.jianshu.io/upload_images/4108852-b38e8cec93e08fbc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-0da8277fcc5838be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们先使用**ServerEndpoint**注解，产生一个WebSocket服务器端及其路径．然后，分别通过**OnOpen, OnClose, OnMessage**注解定义了当有客户端连接到服务器，当有客户端尝试关闭连接，当客户端向服务器端发送消息时如何进行处理．最后，我们定义了一个**sendMessage**方法，这个方法将被**Kafka Consumer**来使用，通过这个方法向客户端发送消息．

完成了WebSocket服务器端代码之后，我们就需要写**Kafka Consumer**的代码了．我们需要让它订阅某个**Kafka Topic**，并能够通过WebSocket发送数据给客户端，实现代码如下:


![](http://upload-images.jianshu.io/upload_images/4108852-155f0c7776a65d99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4108852-e66f4706b0029662.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的代码很容易理解，就不解释了．

完成了**Kafka Consumer**的代码之后，我们还需要让Tomcat能够运行它．因为Tomcat是一个服务器端软件，所以我们不能通过直接写main方法的方式，让Tomcat运行它．所以我们额外写了这么一个类:


![](http://upload-images.jianshu.io/upload_images/4108852-bb65594d3dfcaa9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

写好了之后，我们还需要让Tomcat知道，我们希望他能运行上面那个类．那么我们如何来实现呢?

我们需要在**web.xml**中，添加一个listener,如下所示:


![](http://upload-images.jianshu.io/upload_images/4108852-7baaa8242f5561e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样服务器端就全部写好了，那么我们还需要一个WebUI来显示接收到的数据，我们在**webapp**目录下，新建一个index.html，其内容为:


![](http://upload-images.jianshu.io/upload_images/4108852-30d0f4fe90bd58da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-7aed29aa35a63a04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其显示出来就是这样:


![](http://upload-images.jianshu.io/upload_images/4108852-05a86e0ba49ec0ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当点击**connect**按钮时，会让客户端连接到服务器端，当点击**close**按钮时，客户端从服务器端断开，当点击**sendMsg**按钮时，客户端会向服务器发送一条消息．

当客户端和服务器端处于连接状态中的话，那么服务器端就可以发送数据给客户端．

代码都完成之后，我们需要将项目打包成war包并部署到Tomcat中，在项目根目录下执行下面的命令:

**mvn package**

打包完成之后，将target目录下生成的war文件拷贝到Tomcat根目录下的**webapp**下，然后启动Tomcat即可，通过执行Tomcat bin目录下的**startup.sh**脚本来启动服务器．

一切都做好之后，向Kafka Stream传送数据，等到其处理好之后，就可以在上面的Web UI中看到WebSocket传送来的数据了，如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-b2ed41ff8f361daa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4108852-3d7b9de111e80aad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样就完成了．

由于我的代码是从Github上找的，这里贴出原项目的地址:

[webSocketkafka项目源码](https://github.com/youtNa/webSocketkafka)
