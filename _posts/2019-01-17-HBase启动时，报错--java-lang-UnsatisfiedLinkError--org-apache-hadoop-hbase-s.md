---
layout: post
title: HBase启动时，报错--java-lang-UnsatisfiedLinkError--org-apache-hadoop-hbase-s
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
今天，在打算启动HBase，打断点调试的时候，遇到了这么一个错误:
~~~
017-08-14 16:19:10,522 ERROR [main] hbase.MiniHBaseCluster(230): Error starting cluster
java.lang.RuntimeException: Failed construction of Master: class org.apache.hadoop.hbase.master.HMasterorg.apache.hadoop.hbase.shaded.io.netty.channel.epoll.NativeStaticallyReferencedJniMethods.epollin()I
at org.apache.hadoop.hbase.util.JVMClusterUtil.createMasterThread(JVMClusterUtil.java:143)
at org.apache.hadoop.hbase.LocalHBaseCluster.addMaster(LocalHBaseCluster.java:217)
at org.apache.hadoop.hbase.LocalHBaseCluster.<init>(LocalHBaseCluster.java:152)
at org.apache.hadoop.hbase.MiniHBaseCluster.init(MiniHBaseCluster.java:215)
at org.apache.hadoop.hbase.MiniHBaseCluster.<init>(MiniHBaseCluster.java:94)
at org.apache.hadoop.hbase.HBaseTestingUtility.startMiniHBaseCluster(HBaseTestingUtility.java:1130)
at org.apache.hadoop.hbase.HBaseTestingUtility.startMiniCluster(HBaseTestingUtility.java:1083)
at org.apache.hadoop.hbase.HBaseTestingUtility.startMiniCluster(HBaseTestingUtility.java:954)
at org.apache.hadoop.hbase.HBaseTestingUtility.startMiniCluster(HBaseTestingUtility.java:936)
at org.apache.hadoop.hbase.HBaseTestingUtility.startMiniCluster(HBaseTestingUtility.java:918)
at org.apache.hadoop.hbase.client.AbstractTestScanCursor.setUpBeforeClass(AbstractTestScanCursor.java:75)
at org.apache.hadoop.hbase.client.TestAsyncResultScannerCursor.setUpBeforeClass(TestAsyncResultScannerCursor.java:34)
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:498)
at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
at org.junit.internal.runners.statements.RunBefores.evaluate(RunBefores.java:24)
at org.junit.internal.runners.statements.RunAfters.evaluate(RunAfters.java:27)
at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
at org.eclipse.jdt.internal.junit4.runner.JUnit4TestReference.run(JUnit4TestReference.java:86)
at org.eclipse.jdt.internal.junit.runner.TestExecution.run(TestExecution.java:38)
at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:459)
at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:678)
at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:382)
at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:192)
Caused by: java.lang.UnsatisfiedLinkError: failed to load the required native library
at org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.Epoll.ensureAvailability(Epoll.java:78)
at org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.EpollEventLoopGroup.<clinit>(EpollEventLoopGroup.java:38)
at org.apache.hadoop.hbase.util.NettyEventLoopGroupConfig.<init>(NettyEventLoopGroupConfig.java:61)
at org.apache.hadoop.hbase.regionserver.HRegionServer.<init>(HRegionServer.java:553)
at org.apache.hadoop.hbase.master.HMaster.<init>(HMaster.java:469)
at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
at org.apache.hadoop.hbase.util.JVMClusterUtil.createMasterThread(JVMClusterUtil.java:140)
... 27 more
Caused by: java.lang.UnsatisfiedLinkError: org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.NativeStaticallyReferencedJniMethods.epollin()I
at org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.NativeStaticallyReferencedJniMethods.epollin(Native Method)
at org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.Native.<clinit>(Native.java:66)
at org.apache.hadoop.hbase.shaded.io.netty.channel.epoll.Epoll.<clinit>(Epoll.java:33)
~~~

一度抓耳挠腮，不知道是什么意思。

后来突然想起来，第一次启动时，当时是从Github中pull的Master分支下来编译的。然后，才启动成功的。

而现在我checkout了rel/2.1.0这个tag，打算调试这个tag。而没有重新编译，就启动了。

于是乎，重新编译了一遍HBase。现在没问题了。
