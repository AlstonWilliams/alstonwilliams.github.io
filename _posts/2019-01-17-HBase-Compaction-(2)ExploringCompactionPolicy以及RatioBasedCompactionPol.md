---
layout: post
title: HBase-Compaction-(2)ExploringCompactionPolicy以及RatioBasedCompactionPol
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
在这篇文章中，我会介绍，ExploringCompactionPolicy以及RatioBasedCompactionPolicy，是如何选择需要HStoreFile来进行Compaction。

## HBase版本

我采用的HBase版本是rel-2.1.0这个分支。

## RatioBasedCompactionPolicy

我们首先来看RatioBasedCompactionPolicy。关于它选择HStoreFile的过程，在[HBase官网](http://hbase.apache.org/0.94/book/regions.arch.html)有很详细的介绍。

我们这里还是翻译一下。

我们先贴RatioBasedCompactionPolicy中这部分的代码:
~~~
  /**
    * -- Default minor compaction selection algorithm:
    * choose CompactSelection from candidates --
    * First exclude bulk-load files if indicated in configuration.
    * Start at the oldest file and stop when you find the first file that
    * meets compaction criteria:
    * (1) a recently-flushed, small file (i.e. <= minCompactSize)
    * OR
    * (2) within the compactRatio of sum(newer_files)
    * Given normal skew, any newer files will also meet this criteria
    * <p/>
    * Additional Note:
    * If fileSizes.size() >> maxFilesToCompact, we will recurse on
    * compact().  Consider the oldest files first to avoid a
    * situation where we always compact [end-threshold,end).  Then, the
    * last file becomes an aggregate of the previous compactions.
    *
    * normal skew:
    *
    *         older ----> newer (increasing seqID)
    *     _
    *    | |   _
    *    | |  | |   _
    *  --|-|- |-|- |-|---_-------_-------  minCompactSize
    *    | |  | |  | |  | |  _  | |
    *    | |  | |  | |  | | | | | |
    *    | |  | |  | |  | | | | | |
    * @param candidates pre-filtrate
    * @return filtered subset
    */
  protected ArrayList<HStoreFile> applyCompactionPolicy(ArrayList<HStoreFile> candidates,
    boolean mayUseOffPeak, boolean mayBeStuck) throws IOException {
    if (candidates.isEmpty()) {
      return candidates;
    }

    // we're doing a minor compaction, let's see what files are applicable
    int start = 0;
    double ratio = comConf.getCompactionRatio();
    if (mayUseOffPeak) {
      ratio = comConf.getCompactionRatioOffPeak();
      LOG.info("Running an off-peak compaction, selection ratio = " + ratio);
    }

    // get store file sizes for incremental compacting selection.
    final int countOfFiles = candidates.size();
    long[] fileSizes = new long[countOfFiles];
    long[] sumSize = new long[countOfFiles];
    for (int i = countOfFiles - 1; i >= 0; --i) {
      HStoreFile file = candidates.get(i);
      fileSizes[i] = file.getReader().length();
      // calculate the sum of fileSizes[i,i+maxFilesToCompact-1) for algo
      int tooFar = i + comConf.getMaxFilesToCompact() - 1;
      sumSize[i] = fileSizes[i]
        + ((i + 1 < countOfFiles) ? sumSize[i + 1] : 0)
        - ((tooFar < countOfFiles) ? fileSizes[tooFar] : 0);
    }


    while (countOfFiles - start >= comConf.getMinFilesToCompact() &&
      fileSizes[start] > Math.max(comConf.getMinCompactSize(),
          (long) (sumSize[start + 1] * ratio))) {
      ++start;
    }
    if (start < countOfFiles) {
      LOG.info("Default compaction algorithm has selected " + (countOfFiles - start)
        + " files from " + countOfFiles + " candidates");
    } else if (mayBeStuck) {
      // We may be stuck. Compact the latest files if we can.
      int filesToLeave = candidates.size() - comConf.getMinFilesToCompact();
      if (filesToLeave >= 0) {
        start = filesToLeave;
      }
    }
    candidates.subList(0, start).clear();
    return candidates;
  }
~~~

下面采用官网的例子来解释.

对于Compaction，有几个很重要的配置项。我们一一来介绍:

- hbase.store.compaction.ratio Ratio 在选择HStoreFile时有用 (默认是 1.2f).
- hbase.hstore.compaction.min 进行Compaction时，每个HStore至少选择的HStoreFile的数量(默认是 2).
- hbase.hstore.compaction.max 进行Compaction时，每个HStore能够选择的最多的HStoreFile的数量 (默认是 10).
- hbase.hstore.compaction.min.size (bytes) 只要少于阈值，此HStoreFile就会被Compact(默认为`hbase.hregion.memstore.flush.size`，即128 mb).
- hbase.hstore.compaction.max.size 与`hbase.hstore.compaction.min.size`相反,只要大于这个值，就不会被Compact (默认是 Long.MAX_VALUE).
The minor compaction StoreFile selection logic is size based, and selects a file for compaction when the file <= sum(smaller_files) * hbase.hstore.compaction.ratio.

选择Compact的HStoreFile的逻辑是:
- 如果这个HStoreFile的size，小于`
hbase.hstore.compaction.min.size`。那么,congratulation,你中奖了。枪打出头鸟，那就从你开始进行Compact吧。
- 如果这个HStoreFile的size，小于sum(newer_files)，那么也congratulation，你也中奖了。从你开始Compact。

注意，上面的逻辑是二选一。Compact时，会先对这个HStore中的HStoreFile，按照时间排序，然后，找到满足上面任意条件的那个HStoreFile，选择它以及它后面的(hbase.store.compaction.max - 1)个文件，进行Compact。如果没有这么多个，那么就有多少选择多少。

但是，如果它后面的HStoreFile，如果比(hbase.store.compaction.min - 1)还小，那么，不好意思，下次再进行Compact吧，这次我们就先取消了。

还有一点需要注意的是，上面的sum(newer_files)，尽管HBase的注释里是那么写的，但是其实代码里有个陷阱。我们看代码里是怎样计算sum的:
~~~
    final int countOfFiles = candidates.size();
    long[] fileSizes = new long[countOfFiles];
    long[] sumSize = new long[countOfFiles];
    for (int i = countOfFiles - 1; i >= 0; --i) {
      HStoreFile file = candidates.get(i);
      fileSizes[i] = file.getReader().length();
      // calculate the sum of fileSizes[i,i+maxFilesToCompact-1) for algo
      int tooFar = i + comConf.getMaxFilesToCompact() - 1;
      sumSize[i] = fileSizes[i]
        + ((i + 1 < countOfFiles) ? sumSize[i + 1] : 0)
        - ((tooFar < countOfFiles) ? fileSizes[tooFar] : 0);
    }
~~~

我们看到，其实sum并不只是newer_file的size的和。它还要减去index(就是代码里的i)大于(currentHStoreFileIndex +
 hbase.hstore.compaction.max - 1)这些HStoreFile的size的sum。

也就是说，其实sum(newer_file)，指的是，currentHStoreFileIndex 到 (currentHStoreFileIndex + hbase.hstore.compaction.max - 1)这些文件的size的sum.

官方文档里，说的是，sum(smaller_file)。这也是错误的。因为这份文档是0.94的。而我们现在代码是rel-2.1.0这个分支的。

理解了上面这些，官方文档中的例子就很容易理解了。这里我拿一个举例。

假设我们的配置项如下:
hbase.store.compaction.ratio = 1.0f
hbase.hstore.compaction.min = 3 (files)
hbase.hstore.compaction.max = 5 (files)
hbase.hstore.compaction.min.size = 10 (bytes)
hbase.hstore.compaction.max.size = 1000 (bytes)

并且我们有7个HStoreFile，它们的size分别是7,6,5,4,3,2,1 bytes。我们已经对它们排过序，即现在就是从oldest到newest。那么，最终会选择的进行compact的HStoreFile是7,6,5,4,3。为什么呢?

7 --> Yes, because sum(6, 5, 4, 3, 2, 1) * 1.0 = 21. Also, 7 is less than the min-size
6 --> Yes, because sum(5, 4, 3, 2, 1) * 1.0 = 15. Also, 6 is less than the min-size.
5 --> Yes, because sum(4, 3, 2, 1) * 1.0 = 10. Also, 5 is less than the min-size.
4 --> Yes, because sum(3, 2, 1) * 1.0 = 6. Also, 4 is less than the min-size.
3 --> Yes, because sum(2, 1) * 1.0 = 3. Also, 3 is less than the min-size.
2 --> No. Candidate because previous file was selected and 2 is less than the min-size, but the max-number of files to compact has been reached.
1 --> No. Candidate because previous file was selected and 1 is less than the min-size, but max-number of files to compact has been reached.

官方文档是这么解释的。其实，因为7满足条件了，然后总共可以选择`hbase.hstore.compaction.max`个，所以7,6,5,4,3都被选择了。

另外，我们可以看到，文档中，写的是，**7 --> Yes, because sum(6, 5, 4, 3, 2, 1) * 1.0 = 21**。实际上，应该是**7 --> Yes, because sum(6, 5, 4, 3, 2) * 1.0 = 20**。

这里贴上我关于官方文档中三个例子的单元测试，因为在rel-2.1.0中，并没有发现官方文档说的那个单元测试，就自己写了一个:
~~~
package org.apache.hadoop.hbase.regionserver.compactions;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.regionserver.EmptyStoreConfigInformation;
import org.apache.hadoop.hbase.regionserver.HStoreFile;
import org.apache.hadoop.hbase.regionserver.StoreConfigInformation;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.util.ArrayList;

/**
 *
 * @see "http://hbase.apache.org/0.94/book/regions.arch.html"
 * to get more information
 *
 * */
public class TestRatioBasedCompactionPolicy {

    /**
     *
     * Suppose the order is from old to new.
     *
     * */
    ArrayList<HStoreFile> storeFiles = new ArrayList<>();
    MockStoreFileGenerator mockStoreFileGenerator = new MockStoreFileGenerator(TestRatioBasedCompactionPolicy.class);
    RatioBasedCompactionPolicy compactionPolicy = null;

    private void prepareCompactionPolicy() {
        Configuration configuration = new Configuration();
        configuration.set("hbase.hstore.compaction.ratio", "1.0");
        configuration.set("hbase.hstore.compaction.min", "3");
        configuration.set("hbase.hstore.compaction.max", "5");
        configuration.set("hbase.hstore.compaction.min.size", "10");
        configuration.set("hbase.hstore.compaction.max.size", "1000");

        StoreConfigInformation storeConfigInformation = new EmptyStoreConfigInformation();
        compactionPolicy = new RatioBasedCompactionPolicy(configuration, storeConfigInformation);
    }

    private void prepareCandidateStoreFilesForExamplean zhao1() {
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(100));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(50));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(23));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(12));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(12));
    }

    private void prepareCandidateStoreFilesForExample2() {
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(100));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(25));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(12));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(12));
    }

    private void prepareCandidateStoreFilesForExample3() {
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(7));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(6));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(5));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(4));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(3));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(2));
        storeFiles.add(mockStoreFileGenerator.createMockStoreFileBytes(1));
    }

    @Test
    public void testApplyCompactionPolicyForExample1() throws IOException {
        prepareCandidateStoreFilesForExample1();
        prepareCompactionPolicy();
        ArrayList<HStoreFile> toCompact = compactionPolicy.applyCompactionPolicy(storeFiles, false, false);
        for (HStoreFile hsf : toCompact) {
            System.out.println(hsf.getReader().length());
        }
    }

    @Test
    public void testApplyCompactionPolicyForExample2() throws IOException {
        prepareCandidateStoreFilesForExample2();
        prepareCompactionPolicy();
        ArrayList<HStoreFile> toCompact = compactionPolicy.applyCompactionPolicy(storeFiles, false, false);
        for (HStoreFile hsf : toCompact) {
            System.out.println(hsf.getReader().length());
        }
    }

    @Test
    public void testApplyCompactionPolicyForExample3() throws IOException {
        prepareCandidateStoreFilesForExample3();
        prepareCompactionPolicy();
        ArrayList<HStoreFile> toCompact = compactionPolicy.applyCompactionPolicy(storeFiles, false, false);
        for (HStoreFile hsf : toCompact) {
            System.out.println(hsf.getReader().length());
        }
    }
}
~~~

另外，从**RatioBasedCompactionPolicy.applyCompactionPolicy()**这段代码中，其实你是看不到按照`hbase.hstore.compaction.max`来截取一部分HStoreFile来进行compaction的，你只能看到，它只是把前面的更老的不符合条件的清空了。

所以，上面的单元测试，你会看到，结果跟官方文档中，描述的并不相同。

但是，如果你从头开始测试Compaction，你会发现，它确实进行了截取。这部分代码，以后我会在Compaction并发性的单元测试中给出。

## ExploringCompactionPolicy

ExploringCompactionPolicy跟RatioBasedCompactionPolicy有点像，但是也有区别。

我们先上代码:

~~~
  /**
   *
   * Reading:
   *    Like `RatioBasedCompactionPolicy`, but select files by range, not one by one.
   *    Find out the longest (must large than min_compact_file and less than max_compact_size) file sequence
   *        and the smallest total file size
   *        to compact
   *
   * */
  public List<HStoreFile> applyCompactionPolicy(List<HStoreFile> candidates, boolean mightBeStuck,
      boolean mayUseOffPeak, int minFiles, int maxFiles) {
    final double currentRatio = mayUseOffPeak
        ? comConf.getCompactionRatioOffPeak() : comConf.getCompactionRatio();

    // Start off choosing nothing.
    List<HStoreFile> bestSelection = new ArrayList<>(0);
    List<HStoreFile> smallest = mightBeStuck ? new ArrayList<>(0) : null;
    long bestSize = 0;
    long smallestSize = Long.MAX_VALUE;

    int opts = 0, optsInRatio = 0, bestStart = -1; // for debug logging
    // Consider every starting place.
    for (int start = 0; start < candidates.size(); start++) {
      // Consider every different sub list permutation in between start and end with min files.
      for (int currentEnd = start + minFiles - 1;
          currentEnd < candidates.size(); currentEnd++) {
        List<HStoreFile> potentialMatchFiles = candidates.subList(start, currentEnd + 1);

        // Sanity checks
        if (potentialMatchFiles.size() < minFiles) {
          continue;
        }
        if (potentialMatchFiles.size() > maxFiles) {
          continue;
        }

        // Compute the total size of files that will
        // have to be read if this set of files is compacted.
        long size = getTotalStoreSize(potentialMatchFiles);

        // Store the smallest set of files.  This stored set of files will be used
        // if it looks like the algorithm is stuck.
        if (mightBeStuck && size < smallestSize) {
          smallest = potentialMatchFiles;
          smallestSize = size;
        }

        if (size > comConf.getMaxCompactSize(mayUseOffPeak)) {
          continue;
        }

        ++opts;
        if (size >= comConf.getMinCompactSize()
            && !filesInRatio(potentialMatchFiles, currentRatio)) {
          continue;
        }

        ++optsInRatio;
        if (isBetterSelection(bestSelection, bestSize, potentialMatchFiles, size, mightBeStuck)) {
          bestSelection = potentialMatchFiles;
          bestSize = size;
          bestStart = start;
        }
      }
    }
    if (bestSelection.isEmpty() && mightBeStuck) {
      LOG.debug("Exploring compaction algorithm has selected " + smallest.size()
          + " files of size "+ smallestSize + " because the store might be stuck");
      return new ArrayList<>(smallest);
    }
    LOG.debug("Exploring compaction algorithm has selected {}  files of size {} starting at " +
      "candidate #{} after considering {} permutations with {} in ratio", bestSelection.size(),
      bestSize, bestSize, opts, optsInRatio);
    return new ArrayList<>(bestSelection);
  }
~~~

总的来说，**ExploringCompactionPolicy.applyCompactionPolicy()**，会选择HStoreFile的数量最多，并且，总的size最小的HStoreFile序列，来进行Compact。

ExploringCompactionPolicy，还是rel-2.1.0版本中，默认的CompactionPolicy.

还是上面的例子.

假设我们的配置项如下:
hbase.store.compaction.ratio = 1.0f
hbase.hstore.compaction.min = 3 (files)
hbase.hstore.compaction.max = 5 (files)
hbase.hstore.compaction.min.size = 10 (bytes)
hbase.hstore.compaction.max.size = 1000 (bytes)

并且我们有7个HStoreFile，它们的size分别是7,6,5,4,3,2,1 bytes。我们已经对它们排过序，即现在就是从oldest到newest。那么，最终会选择的进行compact的HStoreFile是5,4,3,2,1。为什么呢?

首先，会选择[7, 6, 5]这三个文件。然后，会选择[7,6,5,4]四个文件，因为它比[7,6,5]的长度要长。随后，会选择[7,6,5,4,3]这五个文件，还是一样的道理。

然后，因为要保证长度至少是5，所以，只能选择[6,5,4,3,2]以及[5,4,3,2,1]。而明显后者的totalSize要比前两者小，所以最终的选择是[5,4,3,2,1]

对上面的单元测试，稍加修改，应该就可以看到测试结果。由于我是直接在测试Compaction并发性的单元测试中看得结果，所以这里就先不贴出来了。

## ExploringCompactionPolicy与RatioBasedCompactionPolicy的关系

直接上一张类图:

![](https://upload-images.jianshu.io/upload_images/4108852-d34d138be5e019fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的对比中，其实我们能看到，**ExploringCompactionPolicy**相比**RatioBasedCompactionPolicy**，总是能拿到更小，更适合做Compaction的HStoreFile序列。

## 后记

上面还有一个叫做FIFOCompactionPolicy的，这个我还没仔细看，但是现在在调这个CompactionPolicy的一个bug，日后如果有时间，我会补上。
