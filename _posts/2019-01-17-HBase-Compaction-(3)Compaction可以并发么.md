---
layout: post
title: HBase-Compaction-(3)Compaction可以并发么
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
这个问题之前一直困扰着我。之前一直怀疑，如果可以并行的话，岂不是要乱套。可是如果不能并行的话，那效率岂不是很低。

后来猜测，是不同Region可以并行，相同Region是串行。

然后，事实证明是我错了。

## 环境

HBase rel-2.1.0

Git上并没有这个分支，需要用rel/2.1.0这个tag新建一个出来。

## 解析

在Compaction时，是有一个线程池的。我们来看一下:

~~~
ThreadPoolExecutor pool;
if (selectNow) {
    // compaction.get is safe as we will just return if selectNow is true but no compaction is
    // selected
    pool = store.throttleCompaction(compaction.getRequest().getSize()) ? longCompactions
            : shortCompactions;
} else {
    // We assume that most compactions are small. So, put system compactions into small
    // pool; we will do selection there, and move to large pool if necessary.
    pool = shortCompactions;
}
pool.execute(
        new CompactionRunner(store, region, compaction, tracker, completeTracker, pool, user));
~~~

上面的这段代码，在`CompactSplit`中。

我们可以看到，每个HStore的Compaction，都会根据预估的需要执行的时间，分配到线程池里。所以，其实相同的HStore，在执行Compaction时，也是会并行的。

那我就有疑问了，如何保证Compaction并行的时候不会混乱？

其实很简单，因为Compaction在执行前，需要执行一个选择要进行Compact的HStoreFile的操作，后面就是针对这些HStoreFile进行合并。所以，其实我们只要保证在选择的时候，是线程安全的就好啦。

那怎么做呢？

从`DefaultStoreFileManager`这个类中，我们能够看到。它有这么一个字段:

~~~
/**
 * List of store files inside this store. This is an immutable list that
 * is atomically replaced when its contents change.
 */
private volatile ImmutableList<HStoreFile> storefiles = ImmutableList.of();
~~~

它的含义，从注释中，就很容易看出来，就是这个HStore有哪些HStoreFiles。

另外，在`HStore`中，还有一个很重要的变量，是:
~~~
private final List<HStoreFile> filesCompacting = Lists.newArrayList();
~~~

这个变量，表示当前`HStore`中，有哪些HStoreFile正在被compact。

每次从`storefiles`中，选出来有资格进行compact的HStoreFile时，都会被加到这个`filesCompacting`中。

~~~
/**
 * Adds the files to compacting files. filesCompacting must be locked.
 */
private void addToCompactingFiles(Collection<HStoreFile> filesToAdd) {
    if (CollectionUtils.isEmpty(filesToAdd)) {
        return;
    }
    // Check that we do not try to compact the same StoreFile twice.
    if (!Collections.disjoint(filesCompacting, filesToAdd)) {
        Preconditions.checkArgument(false, "%s overlaps with %s", filesToAdd, filesCompacting);
    }
    filesCompacting.addAll(filesToAdd);
    Collections.sort(filesCompacting, storeEngine.getStoreFileManager().getStoreFileComparator());
}
~~~

而这个方法被调用以前，是会进行同步的:
~~~
synchronized (filesCompacting) {
	......
	addToCompactingFiles(selectedFiles);
}
~~~

这些代码都在`HStore.requestCompaction(int priority,CompactionLifeCycleTracker tracker, User user)`中。各位看官可以自行探索。

所以，我们可以看到，其实并不是选出来有资格作为compact的HStoreFile以后，就将它们从`DefaultStoreFileManager.storefiles`中移除，而是添加到`HStore.filesCompacting`中。

这样就有一个问题，就是如果`DefaultStoreFileManager.storefiles`中没有新增HStoreFile，或者即使新增了，前面的Compact没有完成，那由于选择进行Compact的HStoreFile很可能都是一样的，所以就会导致后面发出来的compact请求其实是无效的。

## 测试代码

~~~
package org.apache.hadoop.hbase.regionserver;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseTestingUtility;
import org.apache.hadoop.hbase.HColumnDescriptor;
import org.apache.hadoop.hbase.HTableDescriptor;
import org.apache.hadoop.hbase.client.ColumnFamilyDescriptor;
import org.apache.hadoop.hbase.client.ColumnFamilyDescriptorBuilder;
import org.apache.hadoop.hbase.master.HMaster;
import org.apache.hadoop.hbase.regionserver.compactions.CompactionRequester;
import org.apache.hadoop.hbase.regionserver.compactions.MockStoreFileGenerator;
import org.apache.hadoop.hbase.testclassification.MediumTests;
import org.apache.hadoop.hbase.testclassification.RegionServerTests;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.zookeeper.KeeperException;
import org.junit.After;
import org.junit.Before;
import org.junit.Rule;
import org.junit.Test;
import org.junit.experimental.categories.Category;
import org.junit.rules.TestName;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentSkipListMap;
import java.util.concurrent.CountDownLatch;

import static org.apache.hadoop.hbase.regionserver.Store.PRIORITY_USER;

@Category({RegionServerTests.class, MediumTests.class})
public class TestConcurrentCompaction {

    @Rule
    public TestName name = new TestName();
    private static final byte[] COLUMN_FAMILY = Bytes.toBytes("fam1");
    private static final byte[] FAMILY = Bytes.toBytes("f1");
    private HTableDescriptor htd = null;
    private static final HBaseTestingUtility UTIL = HBaseTestingUtility.createLocalHTU();
    protected Configuration conf = UTIL.getConfiguration();

    private HRegion r = null;
    private CompactionRequester compactionRequester;


    public TestConcurrentCompaction() {
        super();

        // Local mode
        conf.setBoolean("hbase.testing.nocluster", true);
    }

    @Before
    public void setUp() throws Exception {
        this.htd = UTIL.createTableDescriptor(name.getMethodName());
        if (name.getMethodName().equals("testCompactionSeqId")) {
            UTIL.getConfiguration().set("hbase.hstore.compaction.kv.max", "10");
            UTIL.getConfiguration().set(
                    DefaultStoreEngine.DEFAULT_COMPACTOR_CLASS_KEY,
                    TestCompaction.DummyCompactor.class.getName());
            HColumnDescriptor hcd = new HColumnDescriptor(FAMILY);
            hcd.setMaxVersions(65536);
            this.htd.addFamily(hcd);
        }
        prepareConf();
        this.r = UTIL.createLocalHRegion(htd, null, null);
    }

    @After
    public void tearDown() throws Exception {
        this.r.close();
    }

    @Test
    public void testConcurrentCompaction() throws IOException, KeeperException, InterruptedException {

        addHStoresToHRegion(r);

        // >>>>>>>>>>>>>>>>>>>
        HStore hStore = r.getStore(COLUMN_FAMILY);
        System.out.println(hStore);
        for (HStoreFile hStoreFile : hStore.getStorefiles()) {
            System.out.println(hStoreFile);
        }
        // <<<<<<<<<<<<<<<<<<<

        initCompactionRequestor(r);
        CountDownLatch latch = new CountDownLatch(2);
        TestCompaction.Tracker tracker = new TestCompaction.Tracker(latch);
        compactionRequester.requestCompaction(r, "No reason", PRIORITY_USER, tracker, null);

        Thread.sleep(5 * 1000);

        compactionRequester.requestCompaction(r, "No reason", PRIORITY_USER, tracker, null);

    }

    private void prepareConf() {
        conf.set("hbase.hstore.compaction.ratio", "1.0");
        conf.set("hbase.hstore.compaction.min", "3");
        conf.set("hbase.hstore.compaction.max", "5");
        conf.set("hbase.hstore.compaction.min.size", "10");
        conf.set("hbase.hstore.compaction.max.size", "1000");
        conf.set("hbase.hstore.defaultengine.compactionpolicy.class",
                "org.apache.hadoop.hbase.regionserver.compactions.ExploringCompactionPolicy");
    }

    private void initCompactionRequestor(HRegion hRegion) throws IOException, KeeperException {
        HRegionServer hRegionServer = new HMaster(conf);
        compactionRequester = new CompactSplit(hRegionServer);
    }

    private void addHStoresToHRegion(HRegion hRegion) throws IOException {

        MockStoreFileGenerator mockStoreFileGenerator = new MockStoreFileGenerator(TestCompaction.class);

        ConcurrentSkipListMap<byte[], HStore> stores = new ConcurrentSkipListMap<>(Bytes.BYTES_RAWCOMPARATOR);
        ColumnFamilyDescriptor columnFamilyDescriptor = ColumnFamilyDescriptorBuilder.of(COLUMN_FAMILY);
        HStore hStore = new HStore(hRegion, columnFamilyDescriptor, conf);

        List<HStoreFile> storeFiles = new ArrayList<>();
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(7));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(6));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(5));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(4));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(3));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(2));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(1));
        stores.put(COLUMN_FAMILY, hStore);

        hStore.addStoreFiles(storeFiles);

        hRegion.addStores(stores);
    }
}
~~~

为了保证一个Compact不会很快就完成，导致实际上这两次compact是串行的。我在`HRegion.compact(CompactionContext compaction, HStore store, ThroughputController throughputController, User user)`中加了这么一段代码:
~~~
// >>>>>>>>>>>>>>>>>>>
try {
    for (HStoreFile storeFile : compaction.getRequest().getFiles()) {
        System.out.println(storeFile);
    }
    Thread.sleep(5 * 60 * 1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
// <<<<<<<<<<<<<<<<<<<
~~~

我们可以看一下日志输出:
~~~
第一个compact的日志:
2019-01-11 09:04:05,681 DEBUG [main] compactions.SortedCompactionPolicy(80): Selecting compaction from 7 store files, 0 compacting, 7 eligible, 16 blocking
2019-01-11 09:04:05,686 DEBUG [main] compactions.ExploringCompactionPolicy(130): Exploring compaction algorithm has selected 5  files of size 15 starting at candidate #15 after considering 12 permutations with 12 in ratio
第二个compact的日志:
2019-01-11 09:04:10,700 DEBUG [main] compactions.SortedCompactionPolicy(80): Selecting compaction from 7 store files, 5 compacting, 0 eligible, 16 blocking
2019-01-11 09:04:10,701 DEBUG [main] compactions.ExploringCompactionPolicy(130): Exploring compaction algorithm has selected 0  files of size 0 starting at candidate #0 after considering 0 permutations with 0 in ratio
2019-01-11 09:04:10,701 DEBUG [main] compactions.SortedCompactionPolicy(258): Not compacting files because we only have 0 files ready for compaction. Need 3 to initiate.
~~~
