---
layout: post
title: HBase-Compaction-(1)何时会进行compaction
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
目前在看HBase中跟Compaction相关的代码。

对于Compaction，我们最关心的一点，就是，何时会进行Compaction。因为这会直接影响我们HBase的读写性能。

## 环境

HBase rel-2.1.0

## 解析

HBase会在这三种情况下，进行Compact:
- MemStore flush时
- 检测线程检测到需要进行Compact时
- 主动调用

### MemStore flush

第一种情况，我们可以简单看一下**HRegion.internalFlushcache(WAL wal, long myseqid, Collection<HStore> storesToFlush, MonitoredTask status, boolean writeFlushWalMarker, FlushLifeCycleTracker tracker)**的注释:
~~~
  /**
   * Flush the memstore. Flushing the memstore is a little tricky. We have a lot of updates in the
   * memstore, all of which have also been written to the wal. We need to write those updates in the
   * memstore out to disk, while being able to process reads/writes as much as possible during the
   * flush operation.
   * <p>
   * This method may block for some time. Every time you call it, we up the regions sequence id even
   * if we don't flush; i.e. the returned region id will be at least one larger than the last edit
   * applied to this region. The returned id does not refer to an actual edit. The returned id can
   * be used for say installing a bulk loaded file just ahead of the last hfile that was the result
   * of this flush, etc.
   * @param wal Null if we're NOT to go via wal.
   * @param myseqid The seqid to use if <code>wal</code> is null writing out flush file.
   * @param storesToFlush The list of stores to flush.
   * @return object describing the flush's state
   * @throws IOException general io exceptions
   * @throws DroppedSnapshotException Thrown when replay of WAL is required.
   */
  protected FlushResultImpl internalFlushcache(WAL wal, long myseqid,
      Collection<HStore> storesToFlush, MonitoredTask status, boolean writeFlushWalMarker,
      FlushLifeCycleTracker tracker) throws IOException {
    ......
  }
~~~

这个方法的调用链，最终会检测，是否需要进行compact．

这段代码，在**HStore.updateStorefiles(List<HStoreFile> sfs, long snapshotId)**方法的最后:
~~~
  /**
   * Change storeFiles adding into place the Reader produced by this new flush.
   * @param sfs Store files
   * @param snapshotId
   * @throws IOException
   * @return Whether compaction is required.
   */
  private boolean updateStorefiles(List<HStoreFile> sfs, long snapshotId) throws IOException {
    ......
    return needsCompaction();
  }
~~~

### 检测线程

具体的类是`HRegionServer.CompactionChecker`，它会每隔一段时间检测一下，是否有Region需要进行Compaction．默认是10s检测一次，当然，我们可以用`hbase.server.thread.wakefrequency`这个选项来控制．

### 主动调用

这种就没什么好说的了．无论是hbase shell里面，还是通过Admin API，我们都可以手动进行Compaction．而且，通常在性能调优时，都是更加推荐对于Major Compaction，使用手动的方式，而不是自动的方式．
