---
layout: post
title: YARN源码解析(6)-CapacityScheduler
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
这篇文章的前半部分，我会翻译一篇不错的关于介绍CapacityScheduler各种概念的文章。实际上，也不能算是翻译，我会在其中穿插一些我的理解，并会删减一些内容。

在后面，会介绍从CapacityScheduler的代码层面，是如何分配一个Container的。

为什么要翻译这么一篇文章呢？因为实际上，CapacityScheduler中的概念还是蛮多的。如果不懂这些概念，可能就理解不了CapacityScheduler。

## 概念介绍

#### CAPACITY AND HIERARCHICAL DESIGN

在YARN中，我们可以控制能够调度的最小和最大资源(包括内存和CPU)。

每个NodeManager都会把这台节点上能够分配的CPU和内存资源报告给ResourceManager。然后，在CapacityScheduler中，会将全部NodeManager的可以用于分配的CPU以及内存资源的总和，作为**root**这个队列的能够分配的资源。

说到**root队列**，就不得不介绍CapacityScheduler中各种Queue之间的关系。在CapacitySchduler中，Queue之间的关系，就是树形关系。我们知道，在树这种数据结构中，会有一个根节点，以及一些中间节点，最底下是叶子节点。上面我们提到的**root队列**，就是在这个树形结构中，作为根节点存在的。所以，同样，还有作为中间节点以及叶子节点的Queue存在。

每一个Queue都有一个Capacity。上面我们已经提到了，**root queue**的Capacity就是全部NodeManager的Capacity的总和。我们亦可以通过百分比的方式，为一个中间节点或者叶子节点设置它的Capacity。设置方式如下:
~~~
<property>
  <name>yarn.scheduler.capacity.<queue-path>.capacity</name>
  <value>percentage_of_parent</value>
</property>
~~~

除了使用这种方式设置Capacity，我们也可以为一个Queue设置minimum capacity以及maximum capacity.

那么为什么需要设置这两个属性呢？以及他们都有什么用呢？

你可能以为，在上面我们设置了一个Queue的capacity之后，这个Queue的capacity就一直会是这么大，不会自动伸缩。比如，有两个Queue，我们称之为Queue A以及Queue B。我们分别给它们分配40%以及60%的Capacity。

可能有这么一个场景，Queue A中的Application已经很多了，已经没有Resource来分配新的Application。而Queue B却基本上没有被使用。

如果一个Queue不能被伸缩，那遇到这种场景，该如何处理？很明显，Queue B中的Capacity被浪费了。

好，那我们允许一个Queue伸缩，那么又会有另一个问题。

如果Queue B把它的Capacity中的90%让给了Queue A。而这时候，大量新的Application又被要求分配到Queue B，这时候，该怎么办？

你可能会说，那就把之前让给Queue A的回收回来呗。可是，这岂是你想回收就回收的。就跟借钱的都是孙子，欠钱的都是大爷一样。

为什么呢？因为YARN是一个资源调度系统，它要从全局考虑问题。如果Queue A借了Queue B的那些Capacity，分配的那些Application，经过长时间运行，刚好要结束了，而Queue B需要的时候，就强制停止这些Application，那这些Application就要重新开始运行，如果是一个非常庞大的任务，需要花费几个小时甚至几天的时间来完成，那么，整个系统的效率，是不是就非常低了？

那回收也不是，不回收也不是，那咋整？

我们考虑一下银行里面的做法。在银行里面，当有人贷款时，是不是会先进行风险评估，然后确定最多能贷给这个人多少钱？

Bingo。

那我们也对应的可以设置一下，Queue A最多能借多少Capacity呀。所以，maximum capacity不就来了嘛。

只不过，目前YARN中，如果你想设置，必须手动设置这个值。相当于银行中的风控手动进行风险评估之后，给这个人设置最大可贷款金额。

而我们都知道，现在银行中，都引入了一些自动化的风控措施。那YARN中是否能够做到这一点呢？目前是无法做到的。阿里的Segma似乎能够做到，因为它是基于机器学习算法的。

另一点，从银行的角度，也需要控制，它需要留多少预算进行其他的活动。总不可能把所有钱都贷出去吧。而这个最低预算的阈值，就相当于我们上面提到的minimum capacity。

好，言归正传。我们来看一张Queue的继承图:

![image](http://upload-images.jianshu.io/upload_images/4108852-3e27120b86e5832e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这幅图中，我们可以看到，每一个Queue(除了Root)都是有一个对应的minimum capacity以及maximum capacity的。而且，我们也可以看到，一个Queue的子Queue，他们的minimum capacity的总和，总是100%，即父Queue的minimum capacity。而maximum capacity则没有这个限制。如**Preference**以及**Low**和**High**所示。

#### MINIMUM USER PERCENTAGE AND USER LIMIT FACTOR

**Minimum User Percentage**以及**User Limit Factor**这两个属性，都是用来控制一个用户能够获得的Queue的Capacity的数量的。

**Minimum User Percentage**用于控制一个用户在请求Resource时，能够获得的最小的Resource。例如，如果**minimum user percentage**是**10%**，那么，如果有10个用户同时向这个Queue请求Resource，那么，每个用户都能获得10%。

但是，尽管是**minimum**，但是这个属性实际上可以被打破。如果一个用户就是想要更少的Resource，那么我们就可以忽略**minimum user percentage**这个属性。

**User Limit Factor**这个参数，则恰恰与之相反。它控制的是，一个用户最多能够获得的Resource。

这个参数的值，是前面我们提到的**minimum capacity**的倍数，注意，不是**Minimum User Percentage**，而是Queue的**minimum capacity**。这里的倍数，不一定是整数倍，只要是一个大于0的数字就好。可以是小数，比如0.5。

如果**User Limit Factor**被设置为1，那么就表示，这个用户能够获得的Resource最多就是这个Queue的**minimum capacity**。如果被设置成0.5，那么就表示，这个用户最多能够获得这个Queue的**minimum capacity**的一半。如果你把它设置成大于1，well，如果一个Queue是多个用户共享的，那么你就要考虑一下，是否存在饥饿问题了。

我们来看一张图：

![image](http://upload-images.jianshu.io/upload_images/4108852-812ee9febf0de9bf?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图你可能会看不懂，因为这张图有错误。好吧，只是我（译者）感觉它有错误，原文的作者当然并没有说这张图有错误。

我们可以看到，在上面那张图中，**Min Capacity**以及**Max Capacity**被分成两部分了。而实际上，应当把**Min Capacity**包含在**Max Capacity**里面，这样更合适。否则这张图说不通的。

当我们创建一个Queue的时候，我们需要根据这个Queue中的Application的类型，比如，是计算密集型还是IO密集型，以及这个Queue中预计的Application的数量来确定这个Queue的各个属性。并设置一个小于1的**User Limit Factor**，来防止这个Queue被一个用户霸占的情况。

#### CONTAINER CHURN

**CHURN Container**指的是，Queue能够持续不断地启动以及释放掉的容器。Queue能够重新快速地回到它的minimum capacity，以及能够将它的Capacity公平地分配给每个用户。

而于此相反，那些一直在运行，并且不会被释放的容器，则可能会导致Queue不能接受新的Application。

如果不允许**preemption**，那么资源永远不会被回收回来。所以，如果你发现一个Queue中，有这种Application，就要小心了。考虑把他们放到一个特殊的队列中，给启动这些Application的用户设置**User Limit Factors**，或者允许**preemption**.

#### CPU SCHEDULING (DOMINANT RESOURCE FAIRNESS)

在YARN中，默认情况下，是不允许CPU调度的。有一种叫做Dominant Resource Fairness(DRF)的方式，即选择那个你最常用的调度的资源类型进行调度。

如果按照这种方式，且按照CPU进行调度，那么CPU最终会成为系统的瓶颈。因为一个集群中，一般来说，CPU相对于内存来说，更容易成为瓶颈。

下面我们来看一幅图片，来了解按照CPU进行调度的话，启用的CPU更少。

![image](http://upload-images.jianshu.io/upload_images/4108852-8f4d3fefe560367c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，如果按照内存进行调度，能够运行20个容器，而如果按照CPU进行调度的话，则仅仅只能运行10个容器。

#### PREEMPTION

我们在前面介绍过，可以通过**minimum capacity**以及**maximum capacity**来防止一个Queue饥饿的现象。

可是，如果确实出现了这种现象，那如何解决？

如果启用了**preemption**，那么CapacityScheduler会查找那些刚启动或者已经超过分配给它的Resource的Container，然后将它们kill掉，回收它们的资源，还给原先那个Queue.

另外，需要注意的是，由于**preemption**是跨Queue的，所以不要指望着，一个Queue中，会通过**preemption**的方式来保持各个用户分配到相等的资源。

![image](http://upload-images.jianshu.io/upload_images/4108852-6f95f223135712ad?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另外，需要注意的是，只有被**preemption**的Resource的数量，能够满足一次Resource Request的时候，**preemption**才会发生。而这里还有两个参数，一个是**Total Preemption Per Round**，这个参数用于控制，在一轮**Preemption**的Resource占集群中总的Resource的比例。另一个参数是**Natual Termination Factor**则是一轮**Preemption**的Resource占集群中总的已经分配的Resource的比例。

可以看到，**Natual Termination Factor**的最大值就是**Total Preemption Per Round**。

同时，我们也能看到，如果**Natual Termination Factor**指定的比例，所**preemption**的Resource一直都达不到一次Resource Request所需要的Resource，那么即使你开启了**preemption**选项，那么实际上，它是不起作用的。

另一个需要注意的地方就是，**preemption**只会让一个Queue拥有它的**minimum capacity**。而并不会让它能够拥有它的**maximum capacity**.

#### QUEUE ORDERING POLICIES

CapacitySchduler目前支持两种调度策略: FIFO和Fair。(译者注：实际上，我从官方文档中看到的是，CapacityScheduler仅支持FIFO。)

FIFO即先运行那些提交时间最久的Application。它有一个致命的弊端，就是如果这个Application如果恰好要独占整个Queue的Resource，那么，后面的Application都会被阻塞。

![image](http://upload-images.jianshu.io/upload_images/4108852-b39ba391698aca8f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Fair这里暂时不介绍。

#### DEFAULT QUEUE MAPPING

我们当然可以在提交Application的时候，通过指定Queue的名字来提交到特定的Queue。但是除此之外，我们还可以配置当我们没有指定Queue的名字时，通过一定的映射规则，将我们的Application提交到特定的Queue。有两种映射方法，一种是通过Group name，一种是通过User name。

![](https://upload-images.jianshu.io/upload_images/4108852-a0c150f6ea88d911.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


配置时，需要注意是把Group name还是User name放在前面，因为它总是会寻找最前面的那个。

#### PRIORITY

当我们提交一个Application时，默认情况下，会把它分配给那个有最多可用Capacity的Queue.

但是，我们也可以自己为Queue指定PRIORITY。让具有更高PRIORITY的Queue接受更多的Application.

![image](http://upload-images.jianshu.io/upload_images/4108852-b6298c436901ba95?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中，我们可以看到，即使Ｑueue Ｂ用的Capacity已经比较多了，但是由于相对来说，它的可用Capacity的比例比Queue A多，所以，新的Application还是会优先在Queue B上分配。

#### LABELS

Label，主要用于集群内部的partition。每个Node，都有一个与之相关的Label，就这样划分出了一个个的Partition。每个Partition都是相互独立的。

Label一般是用在集群中，标注有GPU硬件的Node.

也有一类Label比较特殊，即Shared label。标注有这些Label的节点，可以在空闲时，被其他的Application使用。而一旦有那些注明要使用这些Label标注的节点，那么，这些节点中的Application占用的资源就会被回收。

![image](http://upload-images.jianshu.io/upload_images/4108852-2c05a9d56be037d0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### CONTAINER SIZING

很多使用CapacityScheduler的人都不知道，容器能够获得的Resource，是你设定的**minimum allocation**的倍数。例如，如果你设定了**minimum allocation memory**是**1gb**，那么，当你请求一个需要4.5gb内存的容器时，实际上，会给你分配一个内存为5gb的容器。所以，你在设置这个**minimum allocation**的时候，最好考虑到这种情况。并且**maximum allocation**最好是**minimum allocation**的倍数。

![image](http://upload-images.jianshu.io/upload_images/4108852-0235a7da3e2c9902?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Reservation

在CapacityScheduler中，还有Reservation这个概念。那么这是干嘛的呢？

还是为了解决我们前面频繁碰到的那个问题，即，Queue被一个需要很多资源的Application霸占了。

我们可以通过Reservation先保留一部分资源，分配给其他的高优先级的容器。这就是它的作用。

但是需要注意的是，每台NodeManager只能对应一个Reservation.

## 源码实现

我们主要来看如何分配一个容器。

![](https://upload-images.jianshu.io/upload_images/4108852-605ae5b35896a987.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-d3202a50469fe5ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的代码中，我们可以看到，当NodeManager向这个CapacityScheduler发送**NODE_UPDATE**事件时，CapacityScheduler就会分配容器。

我们可以看到，CapacityScheduler中分配容器是被动分配的，并不是主动分配的。

最终，它会调用LeafQueue的**assignContainersOnNode()**方法，这个方法就会分配一个容器。

![](https://upload-images.jianshu.io/upload_images/4108852-226452d9541064ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我并不会细说具体过程。如果你认真读了我之前的文章，并且理解了上面的内容，应该就很容易想到，他的处理过程。无非就是进行安全验证，以及检查是否有足够的资源分配这些。
