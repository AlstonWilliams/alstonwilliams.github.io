---
layout: post
title: Spark HBase Bulkload数据一致性问题
date: 2019-04-25
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近测试环境出现了一个很奇怪的问题，在算报表的时候，发现总是缺了一部分。

而重新跑一遍流程，这些报表又全部都有了。回想起报表是用Sketch计算的，所以可能是流程中写Sketch的过程出了问题。

我们的流程是这个样子，现在程序里，生成Bulkload需要的HFile，然后再调用一个脚本来将这些HFile Bulkload到HBase中。

而其实这中间就隐藏了一个bug，就是如果第一步生成HFile的时候，任务失败了，只生成了部分HFile，而脚本还是会将这些HFile Bulkload到HBase中，并且由于我们任务的状态码是根据最后这个脚本的返回值确定的，所以就会给我们一个假象是，这个任务成功了，生成的数据也是正确的。

我们来看`HBaseRDDFunctions`中相关代码:
~~~
    /**
     * Spark Implementation of HBase Bulk load for wide rows or when
     * values are not already combined at the time of the map process
     *
     * A Spark Implementation of HBase Bulk load
     *
     * This will take the content from an existing RDD then sort and shuffle
     * it with respect to region splits.  The result of that sort and shuffle
     * will be written to HFiles.
     *
     * After this function is executed the user will have to call
     * LoadIncrementalHFiles.doBulkLoad(...) to move the files into HBase
     *
     * Also note this version of bulk load is different from past versions in
     * that it includes the qualifier as part of the sort process. The
     * reason for this is to be able to support rows will very large number
     * of columns.
     *
     * @param tableName                      The HBase table we are loading into
     * @param flatMap                        A flapMap function that will make every row in the RDD
     *                                       into N cells for the bulk load
     * @param stagingDir                     The location on the FileSystem to bulk load into
     * @param familyHFileWriteOptionsMap     Options that will define how the HFile for a
     *                                       column family is written
     * @param compactionExclude              Compaction excluded for the HFiles
     * @param maxSize                        Max size for the HFiles before they roll
     */
    def hbaseBulkLoad(hc: HBaseContext,
                         tableName: TableName,
                         flatMap: (T) => Iterator[(KeyFamilyQualifier, Array[Byte])],
                         stagingDir:String,
                         familyHFileWriteOptionsMap:
                         util.Map[Array[Byte], FamilyHFileWriteOptions] =
                         new util.HashMap[Array[Byte], FamilyHFileWriteOptions](),
                         compactionExclude: Boolean = false,
                         maxSize:Long = HConstants.DEFAULT_MAX_FILE_SIZE):Unit = {
      hc.bulkLoad(rdd, tableName,
        flatMap, stagingDir, familyHFileWriteOptionsMap,
        compactionExclude, maxSize)
    }
~~~

看注释就能明白是怎么一回事了。

注释中说，调用完这个方法，必须立即调用`LoadIncrementalHFiles.doBulkLoad(...)`将生成的HFile load到HBase中。如果是在同一个程序中，这个一致性是没什么问题的。

但是现在我们是在两个程序中，就出现了一致性的问题。
