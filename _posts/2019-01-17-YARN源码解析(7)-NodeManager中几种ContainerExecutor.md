---
layout: post
title: YARN源码解析(7)-NodeManager中几种ContainerExecutor
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在NodeManager中，有三种运行Container的方式，它们分别是:
- DefaultContainerExecutor
- LinuxContainerExecutor
- DockerContainerExecutor

从它们的名字中，我们就能看得出来，默认情况下，一定使用的是**DefaultContainerExecutor**。

而一般情况下，**DefaultContainerExecutor**也确实能够满足我们的需求。

在这篇文章中，我们会首先介绍**DefaultContainerExecutor**，然后简单介绍**LinuxContainerExecutor**和**DockerContainerExecutor**。

## DefaultContainerExecutor

这个ContainerExecutor的实现实际上很简单，就是通过构建一个脚本来执行而已。支持Windows脚本以及Linux脚本。

在ContainerExecutor启动一个Container的过程中，涉及到了三个脚本，它们分别是:
- default_container_executor.sh
- default_container_executor_session.sh
- launch_container.sh

这三个脚本，都是跟Container相关的，所以它们都被放在一个Container所代表的目录结构下。

在NodeManager中，会为每个Application，以及每个Container建立一个对应的目录，在每个Container的目录下，就放置了一些运行这个Container必需的信息。

一般来说，这些目录是位于**/tmp**这个目录下，并且会在一个Application完成后，被删除。减少磁盘空间的消耗。

![Container在磁盘上的表示](https://upload-images.jianshu.io/upload_images/4108852-6538a9d3113f8fa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们分别查看一下，上面我们所说的那三个脚本文件的内容。

**default-container_executor.sh**:
![](https://upload-images.jianshu.io/upload_images/4108852-9d587d115478b0bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，在这个脚本文件的内部，会启动**default_container_executor_session.sh**这个脚本，并将执行结果写入到这个Container的一个名为**Container ID+pid.exitcode**的文件中。

而**default_container_executor_session.sh**这个脚本呢？

![](https://upload-images.jianshu.io/upload_images/4108852-571fc4ee2162c0a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，它主要是启动**launch_container.sh**这个脚本。

而我们可以看到，**launch_container.sh**中，就负责运行相应的Container，也能是MRAppMaster，也可能是Mapper或者Reducer:

![](https://upload-images.jianshu.io/upload_images/4108852-1778fa1d90b4a8f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在**launch_container.sh**中，设置了很多环境变量。

这里因为我查看了一个ApplicationMaster的Container，所以启动的是**MRAppMaster**。

那么，**DefaultContainerExecutor**应该就是首先执行**default_container_executor.sh**这个脚本，对吧？

嗯嗯，没错的。

我们来查看一下代码:

![](https://upload-images.jianshu.io/upload_images/4108852-c0e7fa698a172a19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-88ed94fe189d7772.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际上，**DefaultContainerExecutor**中，实例化的是一个**UnixLocalWrapperScriptBuilder**对象，而这个对象，是**LocalWrapperScriptBuilder**的一个子类，并且在constructor中调用了**LocalWrapperScriptBuilder**的constructor。

![](https://upload-images.jianshu.io/upload_images/4108852-045a45b0e40b5893.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总之，最后**DefaultContainerExecutor**确实就是直接调用了**default_container_executor.sh**这个脚本。

我们可以看到，它的实现实际上非常简单。

同时，我们也可以看到，这个实现有一些问题，即，对于资源隔离做的并不好。全部Container都是由运行NodeManager的那个用户启动的。

## LinuxContainerExecutor

而LinuxContainerExecutor就解决了上面的那个问题。

从代码中，我们可以看到，现在，launch a container时，它是调用了**container-executor**这个程序:

![](https://upload-images.jianshu.io/upload_images/4108852-3e19259920661a0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-11bc66ea9b699ae5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，**container-executor**这个程序是啥东西呢？

在Hadoop的安装目录下，**bin**目录中，你应该就会发现这个程序。

![](https://upload-images.jianshu.io/upload_images/4108852-acf3e34c878d8038.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，可以直接使用这个程序来实例化一个Container，启动一个Container，给这个Container发信号，以及删除这个Container，而且，我们还可以看到，它还支持挂载cgroup，并且我们可以指定运行Container的用户。

**container-executor**这个程序，是用c语言写的。它的源代码在**$HADOOP_SOURCE_CODE_ROOT/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-nodemanager/src/main/native/container-executor/impl**中。

其中最重要的功能，都是在**container-executor.c**这个源文件中。我们还是主要来看它是如何启动一个Container的。

在**launch_container_as_user()**这个函数中，就实现了启动一个Container的功能。

其中最重要的是这么两部分:

![](https://upload-images.jianshu.io/upload_images/4108852-234d5f9d04b5b023.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-d8d9f3d9f4c0e32c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，在上面的代码块中，fork了一个子进程。**fork()**这个函数，我们可以看一下它的介绍。

![](https://upload-images.jianshu.io/upload_images/4108852-497a82a006ef0e6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从文档中，我们可以看到，对于子进程来说，这个函数会返回0，而对于父进程来说，则返回子进程的的pid。

所以，我们可以看到，子进程会执行**execlp()**函数，来运行**launch_container.sh**脚本，启动一个Container。

而父进程则会一直**waitpid()**函数来查看Container的运行状态，一旦运行结束，就将状态码写到特定文件中。

那我们上面提到的，我们可以指定运行这个Container的用户，是如何实现的？

就是这个函数中的**change_user()**实现的啦。

![](https://upload-images.jianshu.io/upload_images/4108852-853922ae197772b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那cgroup又是怎么一回事呢？

cgroup是Linux中实现资源隔离的一种方式，Docker就是基于CGroup以及Namespace实现的。

这部分的代码我没有细看，但是从这个函数中，确实能看到cgroup的身影。

总的来说，**LinuxContainerExecutor**相对于**DefaultContainerExecutor**，它的有点在于，可以手动指定运行Container的用户，并且可以通过cgroup来增强不同的Container之间的隔离性。

当然啦，它也有缺点。毕竟我们手动指定的用户，必须在每个NodeManager上面都存在才行呀，否则运行起来就会出错误。

## DockerContainerExecutor

这个ContainerExecutor，就是把Docker结合进来了。这样做的好处在于，由于Docker是能够保证一个Container使用的资源，不会大于分配给它的资源，所以，不会出现过度使用资源的问题。

这个ContainerExecutor的实现更加简单，就是运行**docker run**命令启动Docker Container，通过**docker inspect**观察Container的状态。

![](https://upload-images.jianshu.io/upload_images/4108852-c863a462b3194aba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里不再多做介绍。

刚开始学习Hadoop的时候，就意识到Container可以用Docker来做，只是没想到YARN中，已经做到这一点了。

## 总结

并不是很清楚为什么LinuxContainerExecutor中，container-executor要用C实现。可能是涉及到的底层操作比较多，比较密集，就用C来实现吧。
