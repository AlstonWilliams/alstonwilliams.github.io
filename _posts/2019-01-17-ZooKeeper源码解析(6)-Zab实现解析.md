---
layout: post
title: ZooKeeper源码解析(6)-Zab实现解析
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
在阅读了Zab的论文<<Zab:High-performance broadcast for primary-backup systems>>之后，总感觉对Zab还是一头雾水．于是，就阅读了ZooKeeper中相关的部分的源码来了解了一下．

在写这篇文章的时候，部分是参考了网上的一篇我认为写的不错的文章:[荐 ZooKeeper的一致性算法赏析](https://my.oschina.net/pingpangkuangmo/blog/778927)

上面的那篇文章中，基本上已经介绍的八九不离十了，如果你读过Zab的源码，但是有的地方不是很了解的话，去读一下那篇文章，应该会有一个更加清晰的认识．

但是，即使上面的那篇文章已经写的不错了，但是有一些必须注意的东西，确实没有提到的．

## Zab中必须要注意的一些地方

Zab是ZooKeeper实现的核心，它有一些地方也是依赖与ZooKeeper的特性的．

比如，ZooKeeper的**操作是幂等性**的特性，让它在处理一些细节的时候，可以做到简单的只需要让客户端重新发送一遍即可，而无需考虑客户端发送来的请求是否已经被处理过．

另外，Zab是通过TCP连接的消息的有序性来保证客户端发送过来的消息的有序性以及Leader和Follower之间的消息的有序性．

倘若没有这两点做保证，很多地方的处理，Zab估计就不会那么洒脱了．

所以，如果我们要通过套用Zab来实现一个分布式应用，我们就必须考虑，这个应用能否做到**幂等性**．如果不满足**幂等性**，我们就得对Zab稍作修改才能使用．

## ZooKeeper中Server的几种状态

在介绍Zab的实现之前，我们必须清楚，ZooKeeper中，Server都有哪几种状态．在Zab的paper中，主要有三种状态:

- **ELECTION state**：server正在寻找一个leader
- **FOLLOWING state**：集群中已经有一个leader了，并且server作为follower正在跟随这个leader
- **LEADING state**：这台server就是那个leader

但是，实际上在ZooKeeper的实现中，Server会处于下面的几种状态：

- **LOOKING state**：对应**ELECTION**状态
- **FOLLOWING state**：对应Zab paper中的**FOLLOWING state**
- **LEADING state**：对应Zab paper中的**LEADING state**
- **OBSERVING state**：是ZooKeeper中自己提出的一种状态，主要对应的是Observer Server．Observer跟Follower的区别在于，它不参与投票，即当Leader统计某个proposal的票数时，是不会把Observer计算在内的．

那么一台Server是如何在上面的四个状态中转换的呢？

当一台Server启动时，它会首先处于**LOOKING**状态，此时会尝试找到一个leader，如果找到，那么就看看这个leader是不是就是它，如果不是的话，就转换为**FOLLOWING/OBSERVING**状态，如果是它，那么就转换为**LEADING**状态．

当处于**FOLLOWING/OBSERVING**状态的server检测到它跟它知道的leader失去联系时，就会重新进入到**LOOKING**状态，并尝试重新找一个leader.

同样，当处于**LEADING**状态的server发现它跟处于**FOLLOWING**状态的server失去联系时，也会进入到**LOOKING**状态并尝试重新找到一个leader．

具体的leader的选举过程，我们会在下文中介绍．

而且，细心地读者可能已经注意到了，在上文中，我们说的是处于**LEADING**状态的server，而不是直接说leader．这是为什么呢？

因为leader并不一定处于**LEADING state**．为什么呢？

因为每台server都会记录当前它认为的leader，这个leader并不一定就是此时集群中的leader．leader也会定时向Follower或Observer发送PING信息，看它是否还是集群中的leader．假如此时集群中的leader发生了故障，并变化为**LOOKING state**，而集群中的Follower或Observer需要隔一段时间才能感知到它认为的leader此时已经跟它失去了联系．

## Zab在ZooKeeper中的实现

在Zab的实现中，主要是经历了这样三个阶段：

**Fast Leader Election ->Synchronization -> Broadcast**

#### Fast Leader Election

从名字中，我们也能看出来，这个阶段就是选择一个leader的．

这部分的源码主要是在**FastLeaderElection.java**的**lookForLeader()**方法中．

那什么时候，才需要选择一个leader呢？

- 当集群启动时，所有的server都处于LOOKING状态
- 当server从故障中恢复时
- 当有新的server想要加入集群时

我们先介绍几个关键变量：
- logicalclock：记录此server当前选举的轮次
- Notification：寻找leader时用到的一个数据结构，在各台Server之间，用于传输选举数据．
- Notification.version：Notification的版本号，主要是为了向后兼容，之前的版本和当前版本的Notification中的某些属性可能不同
- Notification.leader：当前Server想要选举为leader的Server
- Notification.zxid：当前Server记录的它最后收到的proposal的id，**zxid**由两部分组成，一部分是**epoch**，一部分是**proposal id**．其中**epoch**是leader的任期，**transaction id**是就是proposal的id.
- Notification.electionEpoch：当前Server所处的选举轮次
- Notification.state：当前Server的状态，**LOOKING/LEADING/FOLLOWING/OBSERVING**中的一个．
- Notification.sid：当前Server的id
- Notification.peerEpoch：当前Server记录的它选举的leader的任期

这里比较容易混淆的是**electionEpoch**和**peerEpoch**，以及**electionEpoch**和**logicalclock**．

**electionEpoch**和**logicalclock**的区别在于，**electionEpoch**指的是发出Notification的server的**logicalclock**，而**logicalclock**则指的是当前Server所处的选举的轮次，每次调用**lookForLeader()**方法，它的值都会加一．

**electionEpoch**和**peerEpoch**的区别在于，**electionEpoch**记录的选举的轮次，而**peerEpoch**则指的是当前leader的任期．什么意思呢？

想象这样一种情景，一个Server处于网络分区的状态，它会不断调用**lookForLeader()**方法来寻找leader，上面我们也介绍过，每次调用**lookForLeader()**方法，**logicalclock**的值都会加一，所以这台Server就会不断地向外发送**electionEpoch**逐渐增加，而**peerEpoch**不变的notification．直到网络分区恢复，并成功找到了一台leader．

有点难理解？没关系，在下面的选举过程中，你就直到它们都有什么用了．

一台ZooKeeper　Server，假设它的代号为A，在启动时，都会先从本地的Snapshot file以及Transaction logs中读取数据，尽可能地恢复数据．此时，从历史数据中，它能够获取到**latestLoggedZxid**以及**peerEpoch**．

然后，它会将它的sid，以及上面获取到的**latestLoggedZxid**和**peerEpoch**，组装成一个**Notification**，发送给集群中的除**Observer**外的全部成员．注意，发送的对象包括它自己．

其他的Server，这里我们假设有一台Server，其代号为B，在收到来自A的Vote后，通过比较，发现，诶，你小子要选举的leader的数据竟然比我要选举的leader的完整，那我选你选的那个吧，然后记录下从A中收到的**zxid**以及**peerEpoch**，同时，告诉其他Server，我Server B选举Server A选举的那个leader，它的**zxid**为从A中收到的**zxid**，它的**peerEpoch**为从A中收到的**peerEpoch**．

这里有一个问题，就是，Server A想要选举的leader要满足什么条件，Server B才会选举Server要选举的那个leader？

- A的peerEpoch比B的大
- 或者A的peerEpoch跟B的一样大，但是A的zxid比B的大
- 或者A的peerEpoch和zxid都跟B的一样大，但是A的sid比B的大

上面三个条件，满足一个，Server B就会选举Server A要选举的那个leader．

当Server B发现选举Server A选举的那个leader的票数已经超过服务器数的一半了，它就看看是否还有比服务器比Server A选举的那个leader更适合成为leader，如果不存在的话，就进入到**FOLLOWING**状态，成为leader，如果存在的话，就继续上面的过程．直到最后进入**FOLLOWING**状态，或者超时．

这里我们思考一下，有没有可能产生出现两个leader的可能性呢？

假设有五台服务器，并产生了网络分区，Server A和Server B处于一个网络分区，另外三台服务器又处于另一个网络分区中．由于Notification采用的是广播的形式，所以，很可能Server A选举它自己为leader，然后告诉Server B，Server B收到后，通过对比，发现Server B更适合成为leader,然后告诉Server A，Server A收到后，发现确实Server B更适合成为leader，到最后，Server B就到了**LEADING**状态，而Server A就到了**FOLLOWING**状态．

此时，另一个网络分区中，也已经选举出了一个leader．

Server A此时尽管到了**FOLLOWING**状态，但是最终它是不会认为Server B此时是leader的，Server B也成不了leader，为什么呢？在**Synchronization**阶段中，我们会解答这个问题．

最终Server B和Server A只能又回到**LOOKING**状态．

所以，是不可能产生两个leader的，即使发生了网络分区．

另一个问题是，如果Server A发生了网络分区，那么它会无限次调用**lookForLeader()**方法，每次调用这个方法，**logicalclock**都会+1．此时，怎么办呢？

答案是，还是按照之前说的那三个条件来判断是否Server A更适合成为leader，而那三个条件中，没有一个是根据选举轮次**logicalclock**进行判断的．也就是说，其他的Server，比如Server B，在收到长时间分区后来又恢复的Server A的vote时，会先将自己的选举轮次也调整为Server A的选举轮次，然后用上面的三个条件进行比较，选择更适合作为leader的Server．

如果集群中全部的Server此时都是**LOOKING**状态，这样做很正确．但是，如果此时集群中已经存在leader了，有没有可能Server A的数据比此时集群中的leader更新，更适合成为leader呢？

之前这里我纠结了好久，最终发现是不可能的．考虑Server A刚发生分区的时候，此时我们假设Server A确实有一些最新的数据，即Server A是Leader或者Server A已经从Leader中接收到了最新的数据．这里我们只要分清**proposed但是未被commited的proposal**和**proposed并且已经被commited的proposal**的区别，就很容易理解为什么不可能．

实际上，这里还跟Zab处理proposal的过程相关．跟2PC类似，Zab是将Proposal发送给除**Observer**之外的全部**Server**，然后，当收到其中大多数的**Follower**的回复之后，就再发送一个**COMMIT**命令，告诉**Follower**提交这个proposal，收到大多数的Follower的回复之后，然后再告诉Client，请求已经处理完了．

这里需要注意的地方有两点:

- 只有Leader收到大多数的**Follower**的回复之后，才会发送**COMMIT**命令．在Leader发送**COMMIT**命令之前，proposal是处于**proposed但是未被committed**状态的，而在发送**COMMIT**之后，proposal就进入了**proposed并且已经被commited**的状态．

- 只有大多数收到Follower的**COMMIT**回复之后，Leader才告诉Client，请求已经处理完了．

这里我们可以看到，只要Client认为请求已经被处理完，那么肯定有一多半的Follower已经提交了proposal．也就是说，此时，集群中肯定至少有一半的Server的数据跟Server A的数据一样新．

所以，此时选举的时候，选举出的leader肯定包含全部**proposed并且已经被committed**的proposal．

即使丢失了一些**proposed但是未被committed**的proposal，这也无所谓，因为leader并没有发送给客户端请求成功的消息．并且ZooKeeper的操作都是幂等性的，所以，再让客户端发送一遍也无所谓．

这里也就是前面我们说的Zab的实现依赖于ZooKeeper的操作是幂等性的特性．试想，如果ZooKeeper的操作不是幂等性的，这里能这么简单地进行处理吗？

还有一个问题是，由于Leader和Follower有各自的超时机制，Leader如果在一段时间内未收到大多数Follower的响应，就会认为它已经失去了集群的支持，就会进入**LOOKING**状态，而Follower如果在一段时间内未收到来自Leader的**PING**命令，会认为它已经和Leader失去了响应，从而进入**LOOKING**状态，重新查找Leader．

但是，这个时间是物理时间，它在各个机器上并不是严格相等的，中间可能存在一定的误差，即使我们知道只要误差小于一定的值，基本上就可以达到同步，但是Zab的实现中可不是这么做的．这就会导致可能Leader已经超时，进入了**LOOKING**状态，而Follower只能过一段时间才能检测到这一事件，并且也进入**LOOKING**状态．

如果此时有一台Server想要加入集群，此时Follower都会告诉这台Server此时的leader时那台已经宕机的Server．那么如何解决这个问题呢？

所以，在Zab的实现中，有这么一段代码，用来检测Follower告诉这台Server此时的leader是否给这台Server发送过消息，以及Follower认为的leader此时是否确实处于**LEADING**状态：


![](http://upload-images.jianshu.io/upload_images/4108852-a8d596b893ab1d4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Synchronization

在找到一个leader之后，follower必须向leader发送一条**FOLLOWERINFO**信息，向leader表明此follower已经接受过的leader的epoch．

当leader收到大多数follower的**FOLLOWERINFO**之后，它会从中选择一个最大的epoch，然后将其+1作为leader的epoch．

然后，通过**LEADERINFO**命令，leader将新的epoch发送给各个follower．Follower在收到leader的**LEADERINFO**命令之后，会将leader的epoch设置为leader的epoch，并且通过**ACKEPOCH**消息告诉leader该follower的zxid．leader收到**ACKEPOCH**消息之后，会查看follower是否缺少某些proposal logs，如果缺少就发送给follower，如果follower中存在一些leader中没有的proposal logs，那么就告诉follower截断这部分日志．这都是通过**NEWLEADER**命令来做的．

在follower接收到来自leader的**NEWLEADER**命令时，会记录下需要同步的日志，然后回复leader，leader在收到follower的回复之后，就会发送一个COMMIT命令，让follower真正进行同步．

需要注意的是，这部分操作都是同步的，也就是说，在同步日志的时候，follower是无法处理来自客户端的请求的．试想，如果follower此时还处理来自客户端的请求，那日志同步哪时候能做完？

在同步完成后，follower会告诉leader同步已经完成．

此时有两个问题．

第一个问题是，这里leader收到的只是大多数follower的**FOLLOWERINFO**，而不是全部的**FOLLOWERINFO**，就进行了选择，那么会不会剩下的**FOLLOWERINFO**中，存在更新的epoch呢？

答案是，不会．因为leader只有在收到follower都认同该epoch的**ACKEPOCH**回复，才会将自己设置自己的epoch．即使Server B经历了这样的过程，在选举的过程中，它被选举为leader，此时，它收到来自Follower的**FOLLOWERINFO**，并选择了其中的最大的epoch+1作为它的epoch，然后还没等它将这个epoch发送给follower，就挂掉了．Server B的epoch也不会比集群中其他的Server的更大．

还有一个问题，就是，哪时候follower才会有leader不存在的数据呢？

试想下面的Server B，Server B赢得了选举，被选举为leader，此时，它收到了来自Follower的**FOLLOWERINFO**，并选择了其中的最大的epoch+1作为它的epoch，然后它将这个epoch发送给follower，同步完之后，开始接受客户端请求，然后，它收到了客户端的请求1，并添加到了它的proposal logs中了，在它发送给follower之前，它崩溃了．然后Server C赢得了选举，当Server C给集群中的其他Server同步的时候，Server B突然恢复了，并且通过**lookForLeader()**方法发现此时Server C是集群中的leader．在这种情况下，Server B中就存在Server C中leader不存在的数据．

那么，简单的让follower截断这部分leader不存在的数据，是否会有损失数据的风险呢？

同样，托ZooKeeper的操作是幂等性的福，leader中不存在的数据肯定是还没有给客户端成功信息的数据，所以，让客户端再重新发送一次就好了．

#### Broadcast

在这个阶段，server就开始处理客户端请求了．当leader收到一个客户端请求时，会提出一个zxid为当前zxid+1的proposal，然后广播给集群中的followers，当收到大多数follower的回复时，就告诉follower提交该proposal，在收到大多数follower的提交命令的回复时，就会给客户端一个成功信息．

当follower收到一个提交命令时，会同时提交前面收到的proposal．

同时，leader也会不断向follower发送心跳检测信息，检测一下是否还能连接到集群中的多数follower．follower也有一个超时检测，当一段时间内没有收到leader的心跳检测信息时，就进入到**LOOKING**状态，重新选择一个leader．

## 总结

ZooKeeper中Zab的实现中，由于使用TCP连接保证了消息的有序性，又由于ZooKeeper中操作的幂等性，使Zab的实现复杂度大大降低．

同时，ZooKeeper的实现中，还根据是否有**readonlymode.enable**这个系统变量来确定到底是保证CAP中的CP还是AP．如果这个变量为true，那么就会在server发生分区时，启用一个ReadOnlyServer，这会做到AP，但是无法保证一致性．如果这个变量为false，那么在server发生分区时，不会启用ReadOnlyServer，这会保证CP，但是无法保证可用性．

从上面的解析中，我们也可以看到，在处理proposal时，只要多数Follower同意，那么就提交，而且客户端请求是可以到达Follower/Observer的，所以，ZooKeeper实现的是弱一致性模型．
