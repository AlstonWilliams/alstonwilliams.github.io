---
layout: post
title: HBase ZooKeeper NullPointerException
date: 2019-04-12
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

今天在调试客户环境的时候，发现了这么一个很诡异的问题:
~~~
java.lang.NullPointerException
  at org.apache.hadoop.hbase.zookeeper.ZooKeeperWatcher.getMetaReplicaNodes(ZooKeeperWatcher.java:489)
  at org.apache.hadoop.hbase.zookeeper.MetaTableLocator.blockUntilAvailable(MetaTableLocator.java:550)
  ....
  at org.apache.hadoop.hbase.client.HTable.coprocessorService(HTable.java:1785)
  ....
~~~

我们用的HBase版本是cdh5-1.2.0_5.12.1。

从HBase中找到上述代码：
~~~
/**
 * Get the znodes corresponding to the meta replicas from ZK
 * @return list of znodes
 * @throws KeeperException
 */
public List<String> getMetaReplicaNodes() throws KeeperException {
  List<String> childrenOfBaseNode = ZKUtil.listChildrenNoWatch(this, baseZNode);
  List<String> metaReplicaNodes = new ArrayList<String>(2);
  String pattern = conf.get("zookeeper.znode.metaserver","meta-region-server");
  for (String child : childrenOfBaseNode) {
    if (child.startsWith(pattern)) metaReplicaNodes.add(child);
  }
  return metaReplicaNodes;
}
~~~

即上述代码中的`childrenOfBaseNode`是null.我们继续顺着代码往下看，发现最终只不过就是调用的ZooKeeper的获取children节点的代码。用zk-cli.sh可以这样表示:
~~~
ls /hbase
~~~

好，知道问题出在这儿了。我们就开始调试。起初，我们猜测，是HBase设置的`zookeeper.znode.parent`这个属性设置的有问题。此时并没有注意到这其实是HBase client出现的问题。我们以为，是HBase的HRegionServer，由于要用coprocessor来处理数据，所以coprocessor中需要先定位到HRegionServer，然后再读，出现的问题。

其实这种方法经不起推敲，很可笑。因为coprocessor都已经位于HRegionServer上了，怎么可能还需要跟ZooKeeper交互，获取HRegionServer的位置呢？

排除掉这种想法。我们又猜测，是调用coprocessor的client节点，`zookeeper.znode.parent`属性配置错误。

由于我们是在Spark中调用的coprocessor client，而我们又是在客户的环境调试，无法登陆到它们的集群，也就无法上去看是不是那个节点上的hbase-site.xml有问题。

而我们用的hbase-site.xml,其实很混乱。在Spark任务启动的时候，我们通过--jars参数传了一个，然后运行的Jar包中还包含一个，而且这两个hbase-site.xml的内容还不一样。而我们其实并不知道是用的哪个。

幸好Spark的Executor日志中，会打印出来连接的是哪个Zookeeper节点，以及哪个znode.相关片段如下:
~~~
INFO ZooKeeper: Initiating client connection, connectString=host1:2181,host2:2181,host3:2181,baseZnode=/hbase
~~~

从这里面，我们看到host都正确，znode也都正确。那到底是什么问题呢???

此时，我连上host1去看看存不存在这个znode:
~~~
zk-cli.sh --server host1:2181
ls /hbase
~~~

发现是能获取到的。这就纳闷了。

而此时我们又偶然看到日志中有这么一个片段:
~~~
INFO: ZooKeeper - Session establishment complete on server host1: 2181, sessionid=0x111, negotiated timeout=40000
INFO: ZooKeeper - Session establishment complete on server host2: 2181, sessionid=0x111, negotiated timeout=40000
~~~

也就是说，coprocessor client其实只连了host1和host2这两台ZooKeeper Server，来获取HBase的元数据的。

这样子调试范围就小了很多了，然后我又连上host2去看看存不存在这个znode:
~~~
zk-cli.sh --server host2:2181
ls /hbase
~~~
结果是:
~~~
Node does not exist: /hbase
~~~

然后将这个节点从hbase-site.xml的zookeeper.quorum中删掉，就一切正常了。

但是为什么呢？

以前读过ZooKeeper的源码，知道client和ZooKeeper交互的时候，如果是读的话，会随即分配到Follower节点上。而如果此Follower节点挂掉了，会重新分配到其它的Follower节点上。

而这里我们有一点忘记了，就是ZooKeeper并不是强一致性的。所以我们才会检查了host1上存在这个znode，就以为全部的Follower上都存在。而实际上，ZooKeeper是顺序一致性的。这就导致中间可能存在某些时间窗口，host1上存在这个znode，host2上不存在。

正常来说，这个时间窗口应该很小很小，但是，我们客户环境上，一整天了都没有将`/hbase`这个znode同步到host2上。这应该是ZooKeeper的bug。因为，如果一个Follower很长时间都没有跟其它的Follower保持同步，那么这台Follower可能会相对其它Follower少了很多数据，而我们读取数据的时候，如果刚好选择的是这个Follower，那么就会导致获取不到。而ZooKeeper client选择Follower来获取数据的时候，没有过滤掉这些长时间没有保持同步的节点。

这个没有深入研究，因为环境是客户的，没有办法进入看到底发生了什么。
