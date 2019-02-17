---
layout: post
title: 编译与运行Standalone
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
阅读源码，肯定少不了编译和运行这一步。

我选择的源码的版本是Spark 2.4.0-SNAPSHOT这一个版本。

编译的方法很简单，只需要在Spark的源码目录下，运行下面的命令就好了：
~~~
./build/mvn -DskipTests clean package
~~~

编译比较耗时间，占的CPU也较高。所以建议晚上睡觉时，开着电脑让它编译完成。

编译完以后，就可以运行了。这里我们为了调试方便，只是运行的Standalone，这样就不需要额外安装Hadoop的那一套，或者Mesos这些东西。

Standalone的运行方式也很简单。

首先，运行Spark master:
~~~
sbin/start-master.sh
~~~

然后，在其日志中，我们能够看到一个master的url:

![](https://upload-images.jianshu.io/upload_images/4108852-0e73bc43d4e5e45a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们要记住这个url，后面多次要使用。

然后，我们再来启动一个slave节点:

~~~
sbin/start-slave.sh spark://alstonwilliams:7077
~~~

**start-slave.sh**后面跟的是master的url。你应该换成你的。

然后，修改配置文件(位于**conf**目录下)，将**spark-defaults.conf**修改成下面这样子:
~~~
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

# Default system properties included when running spark-submit.
# This is useful for setting default environmental settings.

# Example:
spark.master                     spark://alstonwilliams:7077
spark.eventLog.enabled           true
spark.eventLog.dir               file:///tmp/spark-events
# spark.serializer                 org.apache.spark.serializer.KryoSerializer
# spark.driver.memory              5g
# spark.executor.extraJavaOptions  -XX:+PrintGCDetails -Dkey=value -Dnumbers="one two three"
~~~

其中**spark.master**是告诉Application,如何找到master。**spark.eventLog.enabled**和**spark.eventLog.dir**是配合HistoryServer使用的，如果不设置，Application不会输出日志，我们在HistoryServer中也就看不到我们跑过的Application。

另外，需要注意的是，*spark.eventLog.dir*对应的目录一定要存在，否则HistoryServer启动时会报错的。

好了，上面这些完成以后，通过**sbin/start-history-server.sh**启动一个HistoryServer，我们就可以愉快的玩耍了。

对了，Spark Master WebUI的端口号，默认是**8080**，Spark Worker WebUI的端口号，默认是**8081**。如果你同时还在开发Web应用，那么这两个端口大概率会被占用。我们可以通过修改**conf/spark-env.sh**来设置新的端口。在我的本机上，我分别设置成了**9090**和**9091**:

![](https://upload-images.jianshu.io/upload_images/4108852-f247267781f39e9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
