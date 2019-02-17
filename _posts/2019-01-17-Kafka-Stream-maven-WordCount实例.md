---
layout: post
title: Kafka-Stream-maven-WordCount实例
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 项目思考-设计与核心实现
tags:
- 项目思考-设计与核心实现
---
我们打算设计一个接口统计系统，根据日志统计出来具有高延时的接口，以及错误信息等．开始打算使用Spark来做，后来得知Kafka中提供了这个功能，叫做Kafka Stream,基本的流处理已经能够实现了．于是就打算直接使用Kafka Stream来做．毕竟结构比较简单．

下面，我将会把操作的步骤，记录下来．

首先，启动ZooKeeper,可以使用Kafka提供的脚本来启动:

**bin/zookeeper-server-start.sh config/zookeeper.properties**

当然，如果你已经安装了ZooKeeper,也可以使用你自己的．

然后，启动Kafka服务器:

**bin/kafka-server-start.sh config/server.properties**

准备好基础环境之后，我们还需要准备一些数据，使用下面的命令:

**echo -e "all streams lead to kafka
hello kafka streams
join kafka summit" > file-input.txt**

创建一个用于存储输入数据的topic:

**bin/kafka-topics.sh --create \
    --zookeeper localhost:2181 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-file-input**
    
将准备的数据添加到我们新创建的topic中:

**bin/kafka-console-producer.sh --broker-list localhost:9092 --topic streams-file-input < file-input.txt**

接下来，我们需要编写代码来统计单词出现的次数．

我们先创建一个maven项目，用IntelliJ创建时，在选择** archetype**的部分中，选择如下图所示的项:


![](http://upload-images.jianshu.io/upload_images/4108852-2cd2bfb0d2adfef4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


创建结束后，我们还需要加入Kafka以及Kafka Stream的依赖，pom文件如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-500328158303048b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


你需要选择你的对应的依赖的版本．我这里由于使用的是Kafka 0.11.0.0,所以我的依赖的版本都是选择的**0.11.0.0**.

我的项目的结构如下:


![](http://upload-images.jianshu.io/upload_images/4108852-021955e2387c15b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图中可以看到，我的代码位于**com.projecthome**包下．其中**App.java**便包含了**WordCount**的代码，代码如下:


![](http://upload-images.jianshu.io/upload_images/4108852-6d5219f972f14469.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


写好之后，打包代码:

**mvn package**

在打包完后，你会在项目的根目录的**target/classes**中发现对应的包和类：


![](http://upload-images.jianshu.io/upload_images/4108852-5b6012cca2cb5607.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后，我们需要进入到**target/classes**目录下，然后执行下面的命令，让Kafka执行我们写的代码:

**bin/kafka-run-class.sh com.projecthome.App**

这条命令永远不会停止，除非你按CTRL+C．但是，如果没有错误信息输出，基本上就成功了．然后，我们通过下面这条命令来查看统计的结果:

**bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic streams-wordcount-output \
    --from-beginning \
    --formatter kafka.tools.DefaultMessageFormatter \
    --property print.key=true \
    --property print.value=true \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer**
    
如果前面你做的都正确，则上面的命令会输出下面的内容:


![](http://upload-images.jianshu.io/upload_images/4108852-d69f9afa8c70174f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如果你不按CTRL+C，上面的那条命令也会一直执行，不会停止．

对执行过程更加详细的解释，请查看此文:[Play with a Streams Application](https://kafka.apache.org/0110/documentation/streams/quickstart)
