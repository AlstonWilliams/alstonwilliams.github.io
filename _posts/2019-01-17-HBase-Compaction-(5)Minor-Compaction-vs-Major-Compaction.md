---
layout: post
title: HBase-Compaction-(5)Minor-Compaction-vs-Major-Compaction
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
以前，我一直都有一个误解，就是以为我们调用`Admin.compact(table)`时，它就一定会进行Minor Compaction。我也曾误认为，调用`Admin.majorCompact(table)`时，一定会进行Major Compaction。

然后，事实并不是这样子。

## 环境

HBase rel-2.1.0

## 解析

我们先来看类图:
![](https://upload-images.jianshu.io/upload_images/4108852-d34d138be5e019fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这里面，`RatioBasedCompactionPolicy`,`ExploringCompactionPolicy`以及`FIFOCompactionPolicy`判断是否尝试majorCompaction的逻辑都是一样的。而`DateTieredCompactionPolicy`则更加复杂。同样,`DateTieredCompactionPolicy`的逻辑我也还没理解，所以这篇文章中，不会介绍`DateTieredCompactionPolicy`。以后等我理解了，会专门写一系列文章来介绍它。

HBase在进行Compaction时，满足下面这个条件时，会将minorCompaction转化成majorCompaction：
- 这次minorCompaction选择了全部的HStoreFiles(这是前提条件，即使你调用`Admin.majorCompaction(table)`，但是不满足这个条件，依然会执行Minor Compaction)
- 选择出来的HStoreFiles的minTimestamp(代表这堆HStoreFiles中的最小的时间戳)，距离现在已经过去了`hbase.hregion.majorcompaction`(默认是7天，好多文章都说是24小时，其实是不正确的，至少rel-2.1.0中不是这样子)这段时间。

我们直接看代码.

~~~
@Override
public ScanType getScanType(CompactionRequestImpl request) {
    return request.isAllFiles() ? COMPACT_DROP_DELETES : COMPACT_RETAIN_DELETES;
}
~~~

这段代码在`Compactor`中。而`request.isAllFiles()`的定义为:
~~~
@Override
public boolean isAllFiles() {
  return this.isMajor == DisplayCompactionType.MAJOR
      || this.isMajor == DisplayCompactionType.ALL_FILES;
}
~~~

而对`CompactionRequestImpl.isMajor`的设置条件相当严格，我们可以看下代码:

~~~
ArrayList<HStoreFile> filesToCompact = Lists.newArrayList(result.getFiles());
removeExcessFiles(filesToCompact, isUserCompaction, isTryingMajor);
result.updateFiles(filesToCompact);

isAllFiles = (candidateFiles.size() == filesToCompact.size());
result.setOffPeak(!filesToCompact.isEmpty() && !isAllFiles && mayUseOffPeak);
result.setIsMajor(isTryingMajor && isAllFiles, isAllFiles);
~~~

我们再进入`CompactionRequestImpl.setIsMajor()`看一下:
~~~
/**
 * Specify if this compaction should be a major compaction based on the state of the store
 * @param isMajor <tt>true</tt> if the system determines that this compaction should be a major
 *          compaction
 */
public void setIsMajor(boolean isMajor, boolean isAllFiles) {
  assert isAllFiles || !isMajor;
  this.isMajor = !isAllFiles ? DisplayCompactionType.MINOR
      : (isMajor ? DisplayCompactionType.MAJOR : DisplayCompactionType.ALL_FILES);
}
~~~

我们可以看到，只要没有选择全部HStoreFile，那么就对不起，我们不能进行Compaction。

那么，什么时候才能选择全部的HStoreFile呢?

这段代码在`SortedCompactionPolicy.getCurrentEligibleFiles()`中:
~~~
protected ArrayList<HStoreFile> getCurrentEligibleFiles(ArrayList<HStoreFile> candidateFiles,
                                                        final List<HStoreFile> filesCompacting) {
  // candidates = all storefiles not already in compaction queue
  if (!filesCompacting.isEmpty()) {
    // exclude all files older than the newest file we're currently
    // compacting. this allows us to preserve contiguity (HBASE-2856)
    HStoreFile last = filesCompacting.get(filesCompacting.size() - 1);
    int idx = candidateFiles.indexOf(last);
    Preconditions.checkArgument(idx != -1);
    candidateFiles.subList(0, idx + 1).clear();
  }
  return candidateFiles;
}
~~~

可以看到，只要这个HStore中，已经有正在进行的Compaction，那么就不能选择全部的HStoreFiles。

所以，总的来说，就是，只要这个HStore中，有正在执行的Compaction，我们就不能运行Major Compaction。而只能进行Minor Compaction。

这是第一个条件。

那么第二个条件呢？即**选择出来的HStoreFiles的minTimestamp(代表这堆HStoreFiles中的最小的时间戳)，距离现在已经过去了`hbase.hregion.majorcompaction`这段时间**。

我们也是直接看代码。

~~~
// Try a major compaction if this is a user-requested major compaction,
// or if we do not have too many files to compact and this was requested as a major compaction
boolean isTryingMajor = (forceMajor && isAllFiles && isUserCompaction)
        || (((forceMajor && isAllFiles) || shouldPerformMajorCompaction(candidateSelection))
        && (candidateSelection.size() < comConf.getMaxFilesToCompact()));
~~~

我们可以看到，对于minorCompaction来说，其实最重要的地方，就在`shouldPerformMajorCompaction(candidateSelection)`这儿。

`RatioBasedCompactionPolicy.shouldPerformMajorCompaction(condidateSelection)`代码如下:
~~~
/*
 * @param filesToCompact Files to compact. Can be null.
 * @return True if we should run a major compaction.
 */
@Override
public boolean shouldPerformMajorCompaction(Collection<HStoreFile> filesToCompact)
  throws IOException {
  boolean result = false;
  long mcTime = getNextMajorCompactTime(filesToCompact);
  if (filesToCompact == null || filesToCompact.isEmpty() || mcTime == 0) {
    return result;
  }
  // TODO: Use better method for determining stamp of last major (HBASE-2990)
  long lowTimestamp = StoreUtils.getLowestTimestamp(filesToCompact);
  long now = EnvironmentEdgeManager.currentTime();
  if (lowTimestamp > 0L && lowTimestamp < (now - mcTime)) {
    String regionInfo;
    if (this.storeConfigInfo != null && this.storeConfigInfo instanceof HStore) {
      regionInfo = ((HStore)this.storeConfigInfo).getRegionInfo().getRegionNameAsString();
    } else {
      regionInfo = this.toString();
    }
    // Major compaction time has elapsed.
    long cfTTL = HConstants.FOREVER;
    if (this.storeConfigInfo != null) {
       cfTTL = this.storeConfigInfo.getStoreFileTtl();
    }
    if (filesToCompact.size() == 1) {
      // Single file
      HStoreFile sf = filesToCompact.iterator().next();
      OptionalLong minTimestamp = sf.getMinimumTimestamp();
      long oldest = minTimestamp.isPresent() ? now - minTimestamp.getAsLong() : Long.MIN_VALUE;
      /**
       *
       * Reading:
       *  One major compaction is executed just a moment ago,        
       *  and some data in this HStoreFile has expired.
       *
       * */
      if (sf.isMajorCompactionResult() && (cfTTL == Long.MAX_VALUE || oldest < cfTTL)) {
        float blockLocalityIndex =
          sf.getHDFSBlockDistribution().getBlockLocalityIndex(
          RSRpcServices.getHostname(comConf.conf, false));
        if (blockLocalityIndex < comConf.getMinLocalityToForceCompact()) {
          LOG.debug("Major compaction triggered on only store " + regionInfo
            + "; to make hdfs blocks local, current blockLocalityIndex is "
            + blockLocalityIndex + " (min " + comConf.getMinLocalityToForceCompact() + ")");
          result = true;
        } else {
          LOG.debug("Skipping major compaction of " + regionInfo
            + " because one (major) compacted file only, oldestTime " + oldest
            + "ms is < TTL=" + cfTTL + " and blockLocalityIndex is " + blockLocalityIndex
            + " (min " + comConf.getMinLocalityToForceCompact() + ")");
        }
      } else if (cfTTL != HConstants.FOREVER && oldest > cfTTL) {
        LOG.debug("Major compaction triggered on store " + regionInfo
          + ", because keyvalues outdated; time since last major compaction "
          + (now - lowTimestamp) + "ms");
        result = true;
      }
    } else {
      LOG.debug("Major compaction triggered on store " + regionInfo
        + "; time since last major compaction " + (now - lowTimestamp) + "ms");
      result = true;
    }
  }
  return result;
}
~~~

很简单，就是判断是否过去了`hbase.hregion.majorcompaction`这段时间，如果没有的话，那么不必多说，直接不执行。而如果确实过去了的话，并且仅有一个HStoreFile，那么需要根据HDFS block来确定是否值得做Major Compaction。如果不仅仅只有一个HStoreFile，那么也不多说，直接梭哈就完事了。

## 测试代码

~~~
package org.apache.hadoop.hbase.regionserver.compactions;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.*;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.regionserver.*;
import org.apache.hadoop.hbase.testclassification.MediumTests;
import org.apache.hadoop.hbase.testclassification.RegionServerTests;
import org.apache.hadoop.hbase.util.*;
import org.junit.BeforeClass;
import org.junit.ClassRule;
import org.junit.Rule;
import org.junit.Test;
import org.junit.experimental.categories.Category;
import org.junit.rules.ExpectedException;

import java.io.IOException;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

@Category({RegionServerTests.class, MediumTests.class})
public class TestMinorCompactionUpgradeToMajorCompaction {

    @ClassRule
    public static final HBaseClassTestRule CLASS_RULE =
            HBaseClassTestRule.forClass(TestMinorCompactionUpgradeToMajorCompaction.class);

    private static final HBaseTestingUtility TEST_UTIL = new HBaseTestingUtility();

    private final byte[] family = Bytes.toBytes("f");

    private final byte[] qualifier = Bytes.toBytes("q");

    @Rule
    public ExpectedException error = ExpectedException.none();

    public HStore getStoreWithName(TableName tableName) {
        MiniHBaseCluster cluster = TEST_UTIL.getMiniHBaseCluster();
        List<JVMClusterUtil.RegionServerThread> rsts = cluster.getRegionServerThreads();
        for (int i = 0; i < cluster.getRegionServerThreads().size(); i++) {
            HRegionServer hrs = rsts.get(i).getRegionServer();
            for (HRegion region : hrs.getRegions(tableName)) {
                return region.getStores().iterator().next();
            }
        }
        return null;
    }

    private HStore prepareData(TableName tableName, boolean usedForMajor2Minor) throws IOException {
        Admin admin = TEST_UTIL.getAdmin();
        TableDescriptor desc = TableDescriptorBuilder.newBuilder(tableName)
                .setValue(DefaultStoreEngine.DEFAULT_COMPACTION_POLICY_CLASS_KEY,
                        RatioBasedCompactionPolicy.class.getName())
                .setValue(HConstants.HBASE_REGION_SPLIT_POLICY_KEY,
                        DisabledRegionSplitPolicy.class.getName())
                .setColumnFamily(ColumnFamilyDescriptorBuilder.newBuilder(family).build())
                .build();
        admin.createTable(desc);
        Table table = TEST_UTIL.getConnection().getTable(tableName);
        TimeOffsetEnvironmentEdge edge =
                (TimeOffsetEnvironmentEdge) EnvironmentEdgeManager.getDelegate();

        int[] sizes;

        if (usedForMajor2Minor) {
            sizes = new int[]{65159, 55159, 45163, 35163, 25167, 15167, 5167};
        } else {
            sizes = new int[]{65159, 55159, 45163, 35163, 25167};
        }

        for (int size : sizes) {
                byte[] value = new byte[size];
                ThreadLocalRandom.current().nextBytes(value);
                table.put(new Put(Bytes.toBytes(size)).addColumn(family, qualifier, value));
            admin.flush(tableName);
            edge.increment(1001);
        }
        return getStoreWithName(tableName);
    }

    @BeforeClass
    public static void setEnvironmentEdge() throws Exception {
        EnvironmentEdge ee = new TimeOffsetEnvironmentEdge();
        EnvironmentEdgeManager.injectEdge(ee);
        Configuration conf = TEST_UTIL.getConfiguration();
        conf.setInt(HStore.BLOCKING_STOREFILES_KEY, 10000);
        conf.set("hbase.store.compaction.ratio", "1.0");
        conf.set("hbase.hstore.compaction.min", "3");
        conf.set("hbase.hstore.compaction.max", "5");
        conf.set("hbase.hregion.majorcompaction", "1000");
        TEST_UTIL.startMiniCluster(1);
    }

    @Test
    public void testMinorCompactionWithoutDelete() throws Exception {
        TableName tableName = TableName.valueOf("1");
        HStore store = prepareData(tableName, false);

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- Before compaction");
        for (HStoreFile hStoreFile : store.getStorefiles()) {
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
        }
        // <<<<<<<<<<<<<<<<<<<

        Thread.sleep(10 * 1000);

        TEST_UTIL.getAdmin().compact(tableName);

        Thread.sleep(10 * 1000);

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- After compaction");
        for (HStoreFile hStoreFile : store.getStorefiles()) {
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
        }
        // <<<<<<<<<<<<<<<<<<<

    }

    // Test even though minor compaction, if it choose all files to compact,
    // then deleted data also be removed
    @Test
    public void testMinorCompactionWithDelete() throws Exception {

        TableName tableName = TableName.valueOf("2");

        HStore store = prepareData(tableName, false);

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- Before compaction");
        for (HStoreFile hStoreFile : store.getStorefiles()) {
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
        }
        // <<<<<<<<<<<<<<<<<<<

        Table table = TEST_UTIL.getConnection().getTable(tableName);


        Delete delete = new Delete(Bytes.toBytes(25167));
        delete.addColumns(family, qualifier);
        table.delete(delete);

        Admin admin = TEST_UTIL.getAdmin();
        // Don't forget flushing table. Otherwise the row is just deleted in MemStore. Not HStoreFiles.
        admin.flush(tableName);

        TEST_UTIL.getAdmin().compact(tableName);

        Thread.sleep(10 * 1000);

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- After compaction");
        for (HStoreFile hStoreFile : store.getStorefiles()) {
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
        }
        // <<<<<<<<<<<<<<<<<<<

    }

    // Test if there is a compaction is executed, and issue major compaction,
    // it also be executed as minor compaction.
    // Add breakpoint in {@link SortedCompactionPolicy.selectCompaction} to see if major compaction or minor compaction is executed.
    @Test
    public void testMajorCompaction2MinorCompaction() throws Exception {
        TableName tableName = TableName.valueOf("3");
        HStore store = prepareData(tableName, true);

        int index = 0;
        List<HStoreFile> filesCompacting = new ArrayList<>();

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- Before compaction");
        List<HStoreFile> allStoreFiles = (List<HStoreFile>) store.getStorefiles();
        for (int i = 0; i < allStoreFiles.size() ; i++) {
            HStoreFile hStoreFile = allStoreFiles.get(i);
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
            if (index < 3) {
                filesCompacting.add(hStoreFile);
            }
            index++;
        }
        // <<<<<<<<<<<<<<<<<<<

        store.addToCompactingFiles(filesCompacting);
        TEST_UTIL.getAdmin().majorCompact(tableName);

        Thread.sleep(5 * 1000);

        // >>>>>>>>>>>>>>>>>>>
        System.out.println("-------- After compaction");
        for (HStoreFile hStoreFile : store.getStorefiles()) {
            System.out.println("--------- " + hStoreFile);
            StoreFileReader storeFileReader = hStoreFile.getReader();
            System.out.println("--------- size: " + storeFileReader.length());
        }
        // <<<<<<<<<<<<<<<<<<<

    }
}
~~~
