---
layout: post
title: HBase-Compaction-(4)Compaction容错性
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
我们一定不想，做一个Compaction，中间失败了，导致我们的数据丢失了。

那HBase的Compaction是如何保证容错性呢?

## 环境

HBase rel-2.1.0

## 解析

其实容错性这儿做的很简单。我们直接上代码:
~~~
List<HStoreFile> sfs = moveCompactedFilesIntoPlace(cr, newFiles, user);
writeCompactionWalRecord(filesToCompact, sfs);
replaceStoreFiles(filesToCompact, sfs);
~~~

上述代码在`HStore.doCompaction(CompactionRequestImpl cr, Collection<HStoreFile> filesToCompact, User user, long compactionStartTime, List<Path> newFiles)`中。

总共有这么三步:
1. 将产生的临时文件写到HDFS对应的地方去
2. 写WAL文件
3. 追加到`StoreFileManager.storefiles`中去，并从`HStore.fileCompacting`中移除选择出来的compact的HStoreFile。

我们来分析每一步出错造成的影响。

假设第一步出错，服务器挂掉了，没关系，由于compact时，生成的文件是保存在tmp下面的，所以没什么影响。

假设第一步没问题，第二步出错了，没有写到WAL文件中。那么，也没关系。由于HRegion启动时，会根据WAL来将HStoreFile添加到`StoreFileManager.storefiles`中去，所以这里相当于没做compact。也不会有数据丢失的影响。最多可能有冗余数据，这里以后可以验证一下。

假设第二步通过，第三步挂掉了。那也没关系。WAL已经写了，启动的时候，根据WAL文件重放就好了。

至于何时会删掉原来用于Compact的HStoreFile，这一点还没看到。不过，从上面的过程中，我们可以看到，不会有数据丢失的风险，最多会有数据冗余的问题。

这一点解决起来应该也不困难，我相信HBase中其实有相应的措施，只是我现在还没看到而已。
