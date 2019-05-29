---
layout: post
title: HBase Bulkload调试过程
date: 2019-05-29
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

## HBase Bulkload调试过程

#### 问题描述
最近跑任务的时候，遇到了几个问题。开始的问题是"NotServingRegionExceptin"，我们把那台RegionServer重启后，这个就OK了。但是呢，又出现了另一个问题，给出的信息是"Call Exception on Row....".

这就有点头大了，我们通过Cloudera的管理界面，看到那个RegionServer确实是启动的，状态没有任何问题，而且端口什么的也没有问题。（其实后来在本机复现了很多次，才想起来应当通过telnet看看哪个端口能不能通，但是此时已经修好了。）

而我观察到的一个现象是, 程序的报错中有如下片段"hosts,60020,1556780778672"，而实际上Cloudera上这台RegionServer的startCode是"1556780778679"。这两个状态不一致。

而startCode其实就是一台RegionServer的启动时间，用于标识不同的RegionServer。所以我初步猜测，会不会是因为ZooKeeper中的server的startCode和HBase meta表中的不一致，导致了HBase在BulkLoad的时候连接不上去，所以导致出错。

Cloudera上RegionServer的startCode是正确的，确实是RegionServer的启动时间。这个是用的ZooKeeper中存储的startCode，而报错信息中的startCode是来自meta表中的.可以通过如下命令查看一个表的Region的startCode:
~~~
scan 'hbase:meta',{FILTER=>"PrefixFilter('table_name,')"}
~~~

## 调试过程

我们先尝试在本机复现，也是让meta表中的startCode和ZooKeeper中的不一致，结果尝试了很多次，BulkLoad没问题，都能Load进去。

然后我们读`LoadIncrementalHFiles`中相关代码，我们发现它定位Region的时候并不需要startCode，不管是不是SecureBulkLoad:
`LoadIncrementalHFiles.tryAtomicRegionLoad`:
~~~
byte[] regionName = getLocation().getRegionInfo().getRegionName();
if (!isSecureBulkLoadEndpointAvailable()) {
    success = ProtobufUtil.bulkLoadHFile(getStub(), famPaths, regionName, assignSeqIds);
}
~~~

`RegionCoprocessorRpcChannel.callExecService`:
~~~
  RegionServerCallable<CoprocessorServiceResponse> callable =
        new RegionServerCallable<CoprocessorServiceResponse>(connection, table, row) {
      @Override
      public CoprocessorServiceResponse call(int callTimeout) throws Exception {
        if (rpcController instanceof PayloadCarryingRpcController) {
          ((PayloadCarryingRpcController) rpcController).setPriority(tableName);
        }
        if (rpcController instanceof TimeLimitedRpcController) {
          ((TimeLimitedRpcController) rpcController).setCallTimeout(callTimeout);
        }
        byte[] regionName = getLocation().getRegionInfo().getRegionName();
        return ProtobufUtil.execService(rpcController, getStub(), call, regionName);
      }
    };
~~~

我们可以看到，它是从meta表中读取RegionName而已，并不需要startCode。

这就很尴尬了，本机复现不出来这个问题。

然后我们只能使用万能的重启大法了。先禁用掉那台RegionServer，然后重启下Master。然后就Ok了。

而且诡异的是，如果我只是禁用掉RegionServer，HBase并没有将那些Region重新分配到一台新的RegionServer。这一点很奇怪.

## 关于PUT

虽然BulkLoad没问题，但是一旦我们把这两个startCode改为不一致的状态,put操作是会有问题的。它会长时间卡住，然后什么错误日志也没有。而改为一致的状态，则什么问题都没有。

具体是什么原因，还没看。

## 总结

BulkLoad时，`LoadIncrementalHFiles`会先读取meta表中的数据，获取到RegionServer的信息并调用对应的RPC。它只需要RegionServer的ip和port，以及RegionName，而跟startCode并没有关系.
