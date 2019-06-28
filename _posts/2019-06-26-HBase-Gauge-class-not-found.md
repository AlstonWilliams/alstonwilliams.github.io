---
layout: post
title: HBase Gauge class not found
date: 2019-06-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---

跑任务的时候遇到了这个问题:
~~~
2019-06-26 10:46:53,058 ERROR [main] client.AsyncProcess (AsyncProcess.java:submit(405)) - Failed to get region location
java.io.IOException: com.google.protobuf.ServiceException: java.lang.NoClassDefFoundError: com/yammer/metrics/core/Gauge
	at org.apache.hadoop.hbase.protobuf.ProtobufUtil.getRemoteException(ProtobufUtil.java:329)
	at org.apache.hadoop.hbase.protobuf.ProtobufUtil.getRowOrBefore(ProtobufUtil.java:1586)
	at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.locateRegionInMeta(ConnectionManager.java:1398)
	at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.locateRegion(ConnectionManager.java:1199)
	at org.apache.hadoop.hbase.client.AsyncProcess.submit(AsyncProcess.java:395)
	at org.apache.hadoop.hbase.client.AsyncProcess.submit(AsyncProcess.java:344)
	at org.apache.hadoop.hbase.client.BufferedMutatorImpl.backgroundFlushCommits(BufferedMutatorImpl.java:238)
	at org.apache.hadoop.hbase.client.BufferedMutatorImpl.flush(BufferedMutatorImpl.java:190)
	at org.apache.hadoop.hbase.client.HTable.flushCommits(HTable.java:1495)
	at org.apache.hadoop.hbase.client.HTable.put(HTable.java:1086)
	at com.hyper.cdp.pandora.saver.SketchHBaseSaver.put(SketchHBaseSaver.java:18)
	at com.hyper.cdp.pandora.split.PackageSplittor.saveSketchToHBase(PackageSplittor.java:229)
	at com.hyper.cdp.pandora.split.PackageSplittor.split(PackageSplittor.java:109)
	at com.hyper.cdp.pandora.ManuallyPackageSplittor.main(ManuallyPackageSplittor.java:12)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:226)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:141)

~~~

经过确认以后，发现是metrics-core.jar不存在。然后导致卡了好久并最终爆出这个错。把这个jar包加上即可解决问题。
