---
layout: post
title: HBase-PrefixTree以及64KB的BLOCKSIZE导致Get阻塞的问题
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
笔者所在的公司，最近遇到了一个非常诡异的问题-我们在执行Get操作时，发现迟迟不能返回，等了好久都超时了。而此时超时时间我们实际上已经设置成了20分钟。

另一个诡异的问题是，我们发现，当去那些超时的RegionServer上看时，发现它们的CPU负载都很高。

## 环境

cdh5-1.2.0_1.12.1

代码可以从[Github](https://github.com/cloudera/hbase)上下载下来，注意切换到对应的分支。

## 准备

首先介绍一下我们这个任务的流程: **Bulk load A -> Get A**

注意上面*A*指的是某张表，我们需要先将新数据bulk load到A中，然后在Get A中的其它历史数据。

我们有如下建表语句:
~~~
create 'test', {METHOD => 'table_att',MAX_FILESIZE => '1073741824',CONFIGURATION => {'REGION_REPLICATION' => '2', 'SPLIT_POLICY'=>'org.apache.hadoop.hbase.regionserver.ConstantSizeRegionSplitPolicy'}},{NAME => 'f1', DATA_BLOCK_ENCODING => 'PREFIX_TREE', BLOOMFILTER => 'ROW', REPLICATION_SCOPE => '0', COMPRESSION => 'SNAPPY', VERSIONS => '1', TTL => '64800000', MIN_VERSIONS => '0', KEEP_DELETED_CELLS => 'FALSE', BLOCKSIZE => '65536', IN_MEMORY => 'false', BLOCKCACHE => 'true'}, {SPLITS_FILE => '/home/alstonwilliams/OpenSourceCode/hbase-cdh/conf/total_splits.txt'}
~~~

其中`total_splits.txt`的内容如下:
~~~
0
1
2
3
4
5
6
7
8
9
A
B
C
D
E
F
G
H
I
J
K
L
M
N
O
P
Q
R
S
T
U
V
W
X
Y
Z
a
b
c
d
e
f
g
h
i
j
k
l
m
n
o
p
q
r
s
t
u
v
w
x
y
z
-
_
~~~

测试数据:
~~~

import com.google.common.base.Joiner
import com.hyper.util.SingleTable
import org.apache.hadoop.hbase.client.{HBaseAdmin, Put}
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.hbase.{HBaseConfiguration, TableName}

object TestPutAtomicSegmentBlockWithKey {

    def main(args: Array[String]): Unit = {

        val tableName = "test"

        val entries = Array(
            new Entry("76", "5", 1543235094947l),
            new Entry("7604", "10", 1543235184060l),
            new Entry("7607", "3", 1543235184060l),
            new Entry("7610", "10", 1543235184055l),
            new Entry("7613", "10", 1543235184055l),
            new Entry("7616", "3", 1543235184055l),
            new Entry("7619", "10", 1543235184055l),
            new Entry("7625", "10", 1543235184055l),
            new Entry("7628", "10", 1543235184055l),
            new Entry("7634", "3", 1543235184055l),
            new Entry("7640", "3", 1543235184055l),
            new Entry("7643", "10", 1543235184055l),
            new Entry("7646", "3", 1543235184055l),
            new Entry("7649", "10", 1543235184055l),
            new Entry("7673", "3", 1543235184055l),
            new Entry("7679", "3", 1543235184055l),
            new Entry("7688", "3", 1543235184055l),
            new Entry("77", "3", 1543235094947l)
        )

        val table = SingleTable.getInstance(tableName)

        for (entry <- entries) {
            val put = new Put(Bytes.toBytes(Joiner.on("").join("1", entry.id)), entry.timestamp)
            put.addColumn(Bytes.toBytes("f1"), Bytes.toBytes(entry.uid), Bytes.toBytes("1"))
            table.put(put)
        }

        val hadmin = new HBaseAdmin(HBaseConfiguration.create())
        hadmin.flush(TableName.valueOf(tableName))

    }

}

case class Entry(id: String, uid: String, timestamp: Long)
~~~

测试代码:
~~~

import java.io.{FileOutputStream, ObjectOutputStream}

import com.google.common.base.Joiner
import com.hyper.util.{RoaringBitMapUtil, SingleTable}
import org.apache.hadoop.hbase.client.Get
import org.apache.hadoop.hbase.util.Bytes

object TestGetSpecificAtomicSegment {

    def main(args: Array[String]): Unit = {

        val tableName = "test"

        val table = SingleTable.getInstance(tableName)
        val get = new Get(Bytes.toBytes(Joiner.on("").join("1", "7610")))
        get.addColumn(Bytes.toBytes("f1"), Bytes.toBytes("10"))
        val result = table.get(get)
        result.getValue(Bytes.toBytes("f1"), Bytes.toBytes("10"))
    }

}

~~~

## 注意

下文中提到的所有`RowKey`这个名词，都不是传统的RowKey。这儿指的是HFile的KeyValue中的Key，它的格式是:
~~~
11205/family:column/TIMESTAMP/Type
~~~

## 过程

#### 第一波

刚开始，我们从HBase的这张表的UI页面中，观测到，阻塞的那台RegionServer上，同时存在ExecService(对应Bulk load)以及Get这两种RPC service。

此时，我猜测，是不是由于bulk load没完，就开始了Get，导致的此问题。

但是，从我们Leader那里得知，Bulk load是阻塞的。它完成以后，后面的Get操作才会执行。

后来，突然醒悟，同时存在这两种RPC Service，是由于第一次失败了，第二次重试，导致的。

#### 第二波

既然我们知道Get操作卡在那儿了。那我们就需要确认为什么会卡在那儿。所以打印出来了线程堆栈来看。

~~~
Thread 5017: (state = BLOCKED)
 - sun.misc.Unsafe.park(boolean, long) @bci=0 (Compiled frame; information may be imprecise)
 - java.util.concurrent.locks.LockSupport.park(java.lang.Object) @bci=14, line=175 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt() @bci=1, line=836 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(java.util.concurrent.locks.AbstractQueuedSynchronizer$Node, int) @bci=67, line=870 (Compiled frame)
 - java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(int) @bci=17, line=1199 (Compiled frame)
 - java.util.concurrent.locks.ReentrantLock$NonfairSync.lock() @bci=21, line=209 (Compiled frame)
 - java.util.concurrent.locks.ReentrantLock.lock() @bci=4, line=285 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreScanner.updateReaders() @bci=4, line=693 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.HStore.notifyChangedReadersObservers() @bci=30, line=1116 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.HStore.bulkLoadHFile(org.apache.hadoop.hbase.regionserver.StoreFile) @bci=91, line=840 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.HStore.bulkLoadHFile(java.lang.String, long) @bci=96, line=809 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.HRegion.bulkLoadHFiles(java.util.Collection, boolean, org.apache.hadoop.hbase.regionserver.Region$BulkLoadListener) @bci=649, line=5429 (Interpreted frame)
 - org.apache.hadoop.hbase.security.access.SecureBulkLoadEndpoint$1.run() @bci=132, line=290 (Interpreted frame)
 - org.apache.hadoop.hbase.security.access.SecureBulkLoadEndpoint$1.run() @bci=1, line=274 (Interpreted frame)
 - java.security.AccessController.doPrivileged(java.security.PrivilegedAction, java.security.AccessControlContext) @bci=0 (Compiled frame)
 - javax.security.auth.Subject.doAs(javax.security.auth.Subject, java.security.PrivilegedAction) @bci=42, line=360 (Interpreted frame)
 - org.apache.hadoop.security.UserGroupInformation.doAs(java.security.PrivilegedAction) @bci=14, line=1897 (Interpreted frame)
 - org.apache.hadoop.hbase.security.access.SecureBulkLoadEndpoint.secureBulkLoadHFiles(com.google.protobuf.RpcController, org.apache.hadoop.hbase.protobuf.generated.SecureBulkLoadProtos$SecureBulkLoadHFilesRequest, com.google.protobuf.RpcCallback) @bci=409, line=274 (Interpreted frame)
 - org.apache.hadoop.hbase.protobuf.generated.SecureBulkLoadProtos$SecureBulkLoadService.callMethod(com.google.protobuf.Descriptors$MethodDescriptor, com.google.protobuf.RpcController, com.google.protobuf.Message, com.google.protobuf.RpcCallback) @bci=78, line=4631 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.HRegion.execService(com.google.protobuf.RpcController, org.apache.hadoop.hbase.protobuf.generated.ClientProtos$CoprocessorServiceCall) @bci=262, line=7859 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.RSRpcServices.execServiceOnRegion(org.apache.hadoop.hbase.regionserver.Region, org.apache.hadoop.hbase.protobuf.generated.ClientProtos$CoprocessorServiceCall) @bci=11, line=1968 (Interpreted frame)
 - org.apache.hadoop.hbase.regionserver.RSRpcServices.execService(com.google.protobuf.RpcController, org.apache.hadoop.hbase.protobuf.generated.ClientProtos$CoprocessorServiceRequest) @bci=26, line=1950 (Interpreted frame)
 - org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(com.google.protobuf.Descriptors$MethodDescriptor, com.google.protobuf.RpcController, com.google.protobuf.Message) @bci=137, line=33652 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcServer.call(com.google.protobuf.BlockingService, com.google.protobuf.Descriptors$MethodDescriptor, com.google.protobuf.Message, org.apache.hadoop.hbase.CellScanner, long, org.apache.hadoop.hbase.monitoring.MonitoredRPCHandler) @bci=59, line=2183 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.CallRunner.run() @bci=366, line=112 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run(org.apache.hadoop.hbase.ipc.CallRunner) @bci=25, line=205 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run() @bci=17, line=163 (Compiled frame)
Thread 5026: (state = IN_JAVA)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.column.ColumnReader.populateBuffer(int) @bci=135, line=81 (Compiled frame; information may be imprecise)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArrayScanner.populateFamily() @bci=22, line=454 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArrayScanner.populateNonRowFields(int) @bci=6, line=440 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArrayScanner.populateNonRowFieldsAndCompareTo(int, org.apache.hadoop.hbase.Cell) @bci=2, line=422 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArraySearcher.positionAtQualifierTimestamp(org.apache.hadoop.hbase.Cell, boolean) @bci=23, line=219 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArraySearcher.positionAtOrAfter(org.apache.hadoop.hbase.Cell) @bci=41, line=124 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.decode.PrefixTreeArraySearcher.seekForwardToOrAfter(org.apache.hadoop.hbase.Cell) @bci=14, line=184 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.PrefixTreeSeeker.seekToOrBeforeUsingPositionAtOrAfter(org.apache.hadoop.hbase.Cell, boolean) @bci=5, line=210 (Compiled frame)
 - org.apache.hadoop.hbase.codec.prefixtree.PrefixTreeSeeker.seekToKeyInBlock(org.apache.hadoop.hbase.Cell, boolean) @bci=3, line=250 (Compiled frame)
 - org.apache.hadoop.hbase.io.hfile.HFileReaderV2$EncodedScannerV2.loadBlockAndSeekToKey(org.apache.hadoop.hbase.io.hfile.HFileBlock, org.apache.hadoop.hbase.Cell, boolean, org.apache.hadoop.hbase.Cell, boolean) @bci=56, line=1348 (Compiled frame)
 - org.apache.hadoop.hbase.io.hfile.HFileReaderV2$AbstractScannerV2.reseekTo(org.apache.hadoop.hbase.Cell) @bci=78, line=617 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreFileScanner.reseekAtOrAfter(org.apache.hadoop.hbase.io.hfile.HFileScanner, org.apache.hadoop.hbase.Cell) @bci=2, line=314 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreFileScanner.reseek(org.apache.hadoop.hbase.Cell) @bci=18, line=226 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.NonLazyKeyValueScanner.doRealSeek(org.apache.hadoop.hbase.regionserver.KeyValueScanner, org.apache.hadoop.hbase.Cell, boolean) @bci=6, line=54 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.KeyValueHeap.generalizedSeek(boolean, org.apache.hadoop.hbase.Cell, boolean, boolean) @bci=151, line=305 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.KeyValueHeap.requestSeek(org.apache.hadoop.hbase.Cell, boolean, boolean) @bci=5, line=261 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreScanner.reseek(org.apache.hadoop.hbase.Cell) @bci=35, line=813 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreScanner.seekAsDirection(org.apache.hadoop.hbase.Cell) @bci=2, line=801 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.StoreScanner.next(java.util.List, org.apache.hadoop.hbase.regionserver.ScannerContext) @bci=820, line=624 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.KeyValueHeap.next(java.util.List, org.apache.hadoop.hbase.regionserver.ScannerContext) @bci=29, line=147 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion$RegionScannerImpl.populateResult(java.util.List, org.apache.hadoop.hbase.regionserver.KeyValueHeap, org.apache.hadoop.hbase.regionserver.ScannerContext, byte[], int, short) @bci=22, line=5735 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion$RegionScannerImpl.nextInternal(java.util.List, org.apache.hadoop.hbase.regionserver.ScannerContext) @bci=396, line=5891 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion$RegionScannerImpl.nextRaw(java.util.List, org.apache.hadoop.hbase.regionserver.ScannerContext) @bci=31, line=5669 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion$RegionScannerImpl.next(java.util.List, org.apache.hadoop.hbase.regionserver.ScannerContext) @bci=40, line=5645 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion$RegionScannerImpl.next(java.util.List) @bci=6, line=5631 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion.get(org.apache.hadoop.hbase.client.Get, boolean) @bci=62, line=6844 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.HRegion.get(org.apache.hadoop.hbase.client.Get) @bci=102, line=6822 (Compiled frame)
 - org.apache.hadoop.hbase.regionserver.RSRpcServices.get(com.google.protobuf.RpcController, org.apache.hadoop.hbase.protobuf.generated.ClientProtos$GetRequest) @bci=210, line=2009 (Compiled frame)
 - org.apache.hadoop.hbase.protobuf.generated.ClientProtos$ClientService$2.callBlockingMethod(com.google.protobuf.Descriptors$MethodDescriptor, com.google.protobuf.RpcController, com.google.protobuf.Message) @bci=77, line=33644 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcServer.call(com.google.protobuf.BlockingService, com.google.protobuf.Descriptors$MethodDescriptor, com.google.protobuf.Message, org.apache.hadoop.hbase.CellScanner, long, org.apache.hadoop.hbase.monitoring.MonitoredRPCHandler) @bci=59, line=2183 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.CallRunner.run() @bci=366, line=112 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run(org.apache.hadoop.hbase.ipc.CallRunner) @bci=18, line=183 (Compiled frame)
 - org.apache.hadoop.hbase.ipc.RpcExecutor$Handler.run() @bci=17, line=163 (Compiled frame)
~~~

当时我们打印的时候，由于是在第一波的基础上，所以能从stack中，看到bulk load操作阻塞，以及Get操作一直在执行。

联想到我们观测到的RegionServer的负载很高，这儿其实就能想到Get操作时，是陷入了死循环。用`top -H -p pid`能看到，确实对应的线程，CPU负载都在100%。

下面贴一个本机复现出来以后截到的图片，并不是上面获取stack的进程的，但是类似：

![](https://upload-images.jianshu.io/upload_images/4108852-3bea4c1350187364.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们此时还认为，是bulk load操作阻塞导致的get有问题(还没从第一波的阴影里反应过来)，而从stack中，我们看到，bulk load是阻塞在请求lock上了，我们来看对应的代码:

~~~
  public void updateReaders() throws IOException {

    lock.lock();
    try {
    if (this.closing) return;

    // All public synchronized API calls will call 'checkReseek' which will cause
    // the scanner stack to reseek if this.heap==null && this.lastTop != null.
    // But if two calls to updateReaders() happen without a 'next' or 'peek' then we
    // will end up calling this.peek() which would cause a reseek in the middle of a updateReaders
    // which is NOT what we want, not to mention could cause an NPE. So we early out here.
    if (this.heap == null) return;

    // this could be null.
    this.lastTop = this.peek();

    //DebugPrint.println("SS updateReaders, topKey = " + lastTop);

    // close scanners to old obsolete Store files
    this.heap.close(); // bubble thru and close all scanners.
    this.heap = null; // the re-seeks could be slow (access HDFS) free up memory ASAP

    // Let the next() call handle re-creating and seeking
    } finally {
      lock.unlock();
    }
  }
~~~
这段代码位于`StoreScanner.updateReaders()`中，从上面的stack中，我们可以看到，bulk load操作最后会调用这里。

而这个lock为什么会被占用掉了呢？为什么bulk load一直获取不到呢?查找源码，发现在Get操作时，当需要读取HStoreFile时，就会先请求这个lock，相关代码如下:
~~~
@Override
  public boolean next(List<Cell> outResult, ScannerContext scannerContext) throws IOException {
    lock.lock();

    try {
    if (scannerContext == null) {
      throw new IllegalArgumentException("Scanner context cannot be null");
    }
    if (checkReseek()) {
      return scannerContext.setScannerState(NextState.MORE_VALUES).hasMoreValues();
    }

    // if the heap was left null, then the scanners had previously run out anyways, close and
    // return.
    if (this.heap == null) {
      close();
      return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
    }
~~~

上面的代码位于`StoreScanner.next(List<Cell> outResult, ScannerContext scannerContext)`中，从stack中，我们可以看到，Get操作会执行到这儿。由于此段代码实在太长，所以只截取了一小部分。

到了这儿，我们才理解事情的缘由:
- 第一次失败时，是由于Get陷入了死循环
- 第二次我们重试时，bulk load在updateReaders时，由于获取不到锁，陷入了阻塞。

Get操作中，有两处死循环，分别如下:
一处位于`PrefixTreeArraySearcher.positionAtOrAfter(Cell key)`:
~~~
  @Override
  public CellScannerPosition positionAtOrAfter(Cell key) {
    reInitFirstNode();
    int fanIndex = -1;

    while(true){
      //detect row mismatch.  break loop if mismatch
      int currentNodeDepth = rowLength;
      int rowTokenComparison = compareToCurrentToken(key);
      if(rowTokenComparison != 0){
        return fixRowTokenMissForward(rowTokenComparison);
      }

      //exact row found, move on to qualifier & ts
      if(rowMatchesAfterCurrentPosition(key)){
        return positionAtQualifierTimestamp(key, false);
      }

      //detect dead end (no fan to descend into)
      if(!currentRowNode.hasFan()){
        if(hasOccurrences()){
          if (rowLength < key.getRowLength()) {
            nextRow();
          } else {
            populateFirstNonRowFields();
          }
          return CellScannerPosition.AFTER;
        }else{
          //TODO i don't think this case is exercised by any tests
          return fixRowFanMissForward(0);
        }
      }

      //keep hunting for the rest of the row
      byte searchForByte = CellUtil.getRowByte(key, currentNodeDepth);
      fanIndex = currentRowNode.whichFanNode(searchForByte);
      if(fanIndex < 0){//no matching row.  return early
        int insertionPoint = -fanIndex - 1;
        return fixRowFanMissForward(insertionPoint);
      }
      //found a match, so dig deeper into the tree
      followFan(fanIndex);
    }
  }
~~~

另一处位于`PrefixTreeArraySearcher.positionAtQualifierTimestamp(org.apache.hadoop.hbase.Cell, boolean)`:
~~~
  protected CellScannerPosition positionAtQualifierTimestamp(Cell key, boolean beforeOnMiss) {
    int minIndex = 0;
    int maxIndex = currentRowNode.getLastCellIndex();
    int diff;
    while (true) {
      int midIndex = (maxIndex + minIndex) / 2;//don't worry about overflow
      diff = populateNonRowFieldsAndCompareTo(midIndex, key);

      if (diff == 0) {// found exact match
        return CellScannerPosition.AT;
      } else if (minIndex == maxIndex) {// even termination case
        break;
      } else if ((minIndex + 1) == maxIndex) {// odd termination case
        diff = populateNonRowFieldsAndCompareTo(maxIndex, key);
        if(diff > 0){
          diff = populateNonRowFieldsAndCompareTo(minIndex, key);
        }
        break;
      } else if (diff < 0) {// keep going forward
        minIndex = currentCellIndex;
      } else {// went past it, back up
        maxIndex = currentCellIndex;
      }
    }

    if (diff == 0) {
      return CellScannerPosition.AT;

    } else if (diff < 0) {// we are before key
      if (beforeOnMiss) {
        return CellScannerPosition.BEFORE;
      }
      if (advance()) {
        return CellScannerPosition.AFTER;
      }
      return CellScannerPosition.AFTER_LAST;

    } else {// we are after key
      if (!beforeOnMiss) {
        return CellScannerPosition.AFTER;
      }
      if (previous()) {
        return CellScannerPosition.BEFORE;
      }
      return CellScannerPosition.BEFORE_FIRST;
    }
  }
~~~

只要这两处有一处陷入死循环，那么，就会导致Get操作完蛋了。

#### 第三波

所以重心就放在调试Get操作上了。

而此时有一个难题，就是我们在生产环境上，每次跑那个任务，都能复现出来。可是，即使我本机有一个HBase，复现起来还是很困难。

首先，要复现，需要本机的HBase，跟生产环境上的HBase，使用的配置一模一样。这个通过把生产环境的hbase-site.xml拷到本机得以实现。

其次，建表语句要和生产环境上**一模一样**。在我们以为是bulk load的问题时，在本机尝试建了一个简单的表，并没有复现出来。而且，由于我们使用了Snappy这种压缩格式，而我又是通过源码编译的HBase，并没有Snappy，所以需要安装这个支持。这个又折腾了一下午。

重点是一模一样，否则如果是配置问题，复现不出来的。

最后，数据也要和生产环境上一模一样。因为当时我们并不清楚是哪个Region出的问题，所以把这张表的全部数据拷到了本地，所幸这张表不大，还能撑的起来。其实这儿完全可以只将出问题的Region导到本地，这样就不用担心数据量的问题了。过程如下:
- 首先从页面上找到出问题的Region，然后找到它的start key以及end key
- 然后，通过HBase自带的导出工具导出:
~~~
hbase org.apache.hadoop.hbase.mapreduce.Export -Dhbase.mapreduce.scan.row.start=0 -Dhbase.mapreduce.scan.row.stop=6 "mytable" "/export/mytable"
~~~
- 然后，将导出的数据，再通过HBase自带的导入工具导入到本机HBase:
~~~
hbase org.apache.hadoop.hbase.mapreduce.Import 'tablename' 'target import location'
~~~

这儿需要注意的是，如果有条件，尽量还是将源表原样导出，即使只导出某个HRegion理论上说也能复现。

好，这样就能在本机复现了。

光尝试在本机复现，又花费了好长时间。

这儿我们又观测到了很诡异的现象:
1. 如果仅仅只是导入进来，而没有flush一下这张表。那么，由于数据是存在于MemStore中的，那么，是可以正常进行Get的。
2. 即使进行了flush，但是如果导入了两次，或者三次(更多次我没试)，每次都进行flush，那么，也是可以正常Get的。开始猜测是由于导入更多次，会写入到Block Cache，或者OS Cache中。但是，从这张表的UI中，我们看到，仅仅读的时候，才能写到Block Cache中。而至于OS Cache呢？我尝试清空了本机的所有OS Cache，也是能够正常Get的。所以，这儿我一直搞不清楚，为什么会出现这种状况。

#### 第五波

在本机复现了，还不够。我们还需要定位具体问题。

首先，我们要缩小数据的范围。我们先定位到导致阻塞的某个具体的RowKey，由于这个表数据较少，这个Region只有一个HFile，所以直接把这个HFile拎出来就好了。如果有多个HFile，那么可以通过
~~~
hbase hfile -e -f file:///tmp/hbase-alstonwilliams/data/default/... | grep rowkey
~~~
定位到具体的HFile，把它拎出来。

上述命令的输出如下:
~~~
K: 110002/f1:3/1544583024242/Put                
K: 110005/f1:10/1544583024242/Put               
K: 110008/f1:3/1544583024242/Put                
K: 110011/f1:10/1544583024242/Put               
K: 110014/f1:3/1544583024242/Put
~~~

拿到这个HFile以后，我们就可以进一步定位到，是哪个block出问题了。我们可以使用如下命令:
~~~
hbase hfile -b -f file:///tmp/hbase-alstonwilliams/data/default/... > block
~~~

其中，需要把读取的HFile换成刚才拎出来的那个。这条命令的输出如下:
~~~
key=11142//LATEST_TIMESTAMP/Maximum             
  offset=3972585, dataSize=80776                    
key=11144//LATEST_TIMESTAMP/Maximum             
  offset=4053361, dataSize=85551                    
key=11148//LATEST_TIMESTAMP/Maximum             
  offset=4138912, dataSize=85289                    
key=1115//LATEST_TIMESTAMP/Maximum              
  offset=4224201, dataSize=86109                    
key=11152//LATEST_TIMESTAMP/Maximum             
  offset=4310310, dataSize=85993                    
key=11154//LATEST_TIMESTAMP/Maximum             
  offset=4396303, dataSize=87282                    
key=11156//LATEST_TIMESTAMP/Maximum             
  offset=4483585, dataSize=85778                    
key=11158//LATEST_TIMESTAMP/Maximum             
  offset=4569363, dataSize=83534                    
key=11163/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=4652897, dataSize=75622                    
key=11165/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=4728519, dataSize=75165                    
key=11167/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=4803684, dataSize=82283                    
key=11171//LATEST_TIMESTAMP/Maximum             
  offset=4885967, dataSize=83581                    
key=11175//LATEST_TIMESTAMP/Maximum             
  offset=4969548, dataSize=83017                    
key=11179//LATEST_TIMESTAMP/Maximum             
  offset=5052565, dataSize=83653                    
key=11183//LATEST_TIMESTAMP/Maximum             
  offset=5136218, dataSize=84271                    
key=11187//LATEST_TIMESTAMP/Maximum             
  offset=5220489, dataSize=72093                    
key=11189/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=5292582, dataSize=68000                    
key=11191/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=5360582, dataSize=67841                    
key=11193/f1:2/LATEST_TIMESTAMP/Maximum         
  offset=5428423, dataSize=80338                    
key=11197//LATEST_TIMESTAMP/Maximum             
  offset=5508761, dataSize=85544                    
key=11201//LATEST_TIMESTAMP/Maximum             
  offset=5594305, dataSize=85472                    
key=11205//LATEST_TIMESTAMP/Maximum 
~~~

我们可以从上面的输出中，定位到具体是哪个block出了问题，然后，我们就可以通过上面的导出工具，将这个block单独导出一下。这样数据的范围一下子就缩小了很多，方便进一步调试。

然后我们新建一张表，仅仅导入刚才我们拎出来的这个block就好了。然后就可以开始调试了。

此时运行Get操作的demo，就会发现，复现成功了。

感兴趣的朋友完全可以通过我在前面给到的测试数据那个程序来进行复现。效果都是一样的。

此时突然想到，会不会是`PrefixTree`这种BlockEncoding有问题，于是尝试换了一下其它的，换成`FAST_DIFF`，重新跑了一下，丫的，竟然成功了.....

幸福就是来的这么突然。

赶紧到生产环境上试一下，也没有问题。

那么，这儿就有一个问题，我们当初为什么要采用`PrefixTree`这种BlockEncoding？跟其它的相比有什么优劣势？对数据读取，写入有什么影响？对Block Cache有什么影响？

在下班的路上，想到，是不是PrefixTree在压缩这段数据时，由于这个block存在了这些特定的RowKey，导致的问题。

这儿有两种测试方式。

第一种是改变Block size，因为改变了blcok size以后，那些Row可能就会被分到其它Block去了。这种方式我在本机用原始数据测试过，改变了Block Size以后，确实Get操作没问题。但是用我上面给的测试数据不行，因为测试数据里只包含原始数据的RowKey，value是没有的。我们原始数据的value是RoaringBitmap，这儿为了方便，只提供了RowKey。

第二种是删掉这个block中，出事的RowKey前面的RowKey。这种很容易操作，只需要在执行上面的测试数据时，把`new Entry("7610", "10", 1543235184055l)`前面的三条排除掉就好了。我们可以发现，这样复现一下，Get操作是没问题的。

所以，其实复现这个问题最难的点是，只有某些RowKey同时存在于同一个block时，才会出现那个问题。

#### 第六波

上面的测试数据，我只给了RowKey，而没有给真实的value。

问题是，我怎么知道不给真实的value能复现呢?

所以第六波操作就是回答这个问题。

我简单了解了一下`PrefixTree`这种Block Encoding，发现它仅仅只是对RowKey进行压缩。详情请点击[Cloudera官网上的这篇文章](https://archive.cloudera.com/cdh5/cdh/5/hbase-0.98.6-cdh5.3.8/book/compression.html)。所以，我尝试了一下，将原始的value替换一下，发现也是可以复现此问题的。

## 结论

由于还有其它的事情要做，确定了问题以后，我就没继续往下调了。

结果就是，HBase cdh5-1.2.0_1.12.1中的PrefixTree有Bug，我编译了HBase cdh5-1.2.0_1.16.1测试过，还是会有同样的问题。

解决方案有两个:
1. 修改Block Encoding，除了PrefixTree，其它的都没有问题
2. 修改Block Size。但是不推荐这种方式，毕竟谁也说不准什么时候特定的rowkey又会跑到一个Block中去。
