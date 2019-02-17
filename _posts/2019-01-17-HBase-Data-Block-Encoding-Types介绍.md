---
layout: post
title: HBase-Data-Block-Encoding-Types介绍
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- HBase
tags:
- HBase
---
本文翻译自[Cloudera HBase官方文档](https://archive.cloudera.com/cdh5/cdh/5/hbase-0.98.6-cdh5.3.8/book/compression.html)

阅读本文前，请了解一下HFile的格式，对阅读本文会大有裨益．

## 简单介绍HFile

我们这里简单介绍一下HFile的组成，让读者知道什么是Data Block Encoding．

HFile中，包含了好几个部分，具体的请查看[HFile Format](https://blog.cloudera.com/blog/2012/06/hbase-io-hfile-input-output/)．我们这里只关心HFile里的Data Block．

HFile在存储每一个Row时，不是把这一条Row的全部Family/Column整合成在一起，保存起来的，如下:
~~~
RowKey | Family:Column1 -> value | Family:Column2 -> value
~~~

它是把这条Row，根据Column拆分成好几个KeyValue，保存起来的，如下:
~~~
RowKey/Family:Column1 -> value
RowKey/Family:Column2 -> value
~~~

我们可以看到，RowKey需要重复保存很多次，而且Family:Column这个往往都是非常相似的，它也需要保存很多次．这对磁盘非常不友好．当Family:Column越多时，就需要占用越多不必要的磁盘空间．

如果仅仅是磁盘空间，也没什么关系，毕竟我们可以通过Snappy/GZ等压缩方式，对HFile进行Compression．而且磁盘又便宜，对吧？

可是，对HBase熟悉的读者，都知道，当读取数据时，是需要先读MemStore，然后再读BlockCache的．那我们的Block越小，能放到BlockCache中的数据就越多，命中率就越高，对Scan就越友好．

Block Encoding就是做这件事情的，它就是通过某种算法，对Data Block中的数据进行压缩，这样Block的Size小了，放到Block Cache中的就多了．

这儿提出两个问题:
- 压缩以后，占的Disk/Memory是少了，但是解压的时候，需要更多的CPU时间．如何均衡呢?
- 如果我们的业务，偏重的是随机Get，那放到Block Cache中不一定好吧？不仅放到Block Cache中的Block很容易读不到，对性能并没有什么提升，还会产生额外的开销，比如将其它偏重Scan的业务的Block排挤出Block Cache，导致其它业务变慢．

下面正儿八经地翻译原文．

## Data Block Encoding Types

HBase中提供了五种Data Block Encoding Types，具体有:
- NONE
- PREFIX
- DIFF
- FAST_DIFF
- PREFIX_TREE

我们一个一个来介绍．`NONE`这种就不介绍了，这个很容易理解．

#### PREFIX

一般来说，同一个Block中的Key(KeyValue中的Key，不仅包含RowKey，还包含Family:Column)，都很相似．它们往往只是最后的几个字符不同．例如，KeyA是`RowKey:Family:Qualifier0`，跟它相邻的下一个KeyB可能是`RowKey:Family:Qualifier1`．

在PREFIX中，相对于NONE，会额外添加一列，表示当前key(KeyB)和它前一个key(KeyA)，相同的前缀的长度(记为PrefixLength)．在上面的例子中，如果KeyA是这个Block中的第一个key，那它的PrefixLength就是0．而KeyB的PrefixLength是23．

很明显，如果相邻Key之间，完全没有共同点，那PREFIX显然毫无用处，还增加了额外的开销．

我们有一些Row，当使用NONE这种Block Encoding时，如下图所示:

![](https://upload-images.jianshu.io/upload_images/4108852-6baa865851c7da42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而如果采用PREFIX这种Block Encoding，那就是这样子了:
![](https://upload-images.jianshu.io/upload_images/4108852-3eb2951ddc883fc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### DIFF

DIFF是对PREFIX的一种改良．它把key看成很多个部分，对每部分进行压缩，提高效率．它添加了两个新的字段，`timestamp`和`type`．如果KeyB的ColumnFamily/key length/value length/type和KeyA相同，那么它就会在KeyB中被省略．

另外，timestamp，存储的是相对于前一个Row的偏移量．

默认情况下，DIFF是不启用的．因为它会导致写数据，以及Scan数据更慢．但是，相对于PREFIX/NONE，它会在Block Cache中缓存更多数据．

用DIFF压缩的block如下图所示:

![](https://upload-images.jianshu.io/upload_images/4108852-c6f37db8472e647e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### FAST_DIFF

FAST_DIFF跟DIFF非常相似，所不同的是，它额外增加了一个字段，表示RowB是否跟RowA完全一样，如果是的话，那数据就不需要重复保存了．

如果在你的场景下，Key很长，或者有很多Column，那么推荐使用FAST_DIFF．

#### PREFIX_TREE

PREFIX_TREE是0.96中引入的．它大致跟PREFIX,DIFF,FAST_DIFF相同，但是它可以让随机读操作，比其它的几种更快．当然，代价是，MemStore写入到HFile时，需要进行更加复杂的Encoding操作，所以会更慢(译者疑问:那Decoding的时候也会更慢啊，而且随机读的话，Block很可能不存在于Block Cache中，那开销主要都在Decoding的时候，所以随机读操作不应该是更慢么?)．

PREFIX_TREE适合于那种Block Cache命中率非常高的场景(译者注:-.-!)．它增加了一个叫做`tree`的字段．这个字段会保存指向这一Row中，全部Cell的索引．这对压缩更加友好．

详情请查看[HBASE-4676](https://issues.apache.org/jira/browse/HBASE-4676)以及[Trie](http://en.wikipedia.org/wiki/Trie)

译者注:我们的HBase用的就是这种Block Encoding，在实际使用的过程中，发现了一个Bug．我们调了好长时间才搞清楚是PREFIX_TREE的问题．详情请查看[HBase PrefixTree以及64KB的BLOCKSIZE导致Get阻塞的问题](https://www.jianshu.com/p/a3a81a9d472c)这篇文章．

## 如何选择压缩算法以及Block Encoding Type?

- 如果Key很长，或者有很多Column，那么推荐使用FAST_DIFF
- 如果数据是冷数据，不经常被访问，那么使用GZIP压缩格式．因为虽然它比Snappy/LZO需要占用更多而CPU，但是它的压缩比率更高，更节省磁盘．
- 如果是热点数据，那么使用Snappy/LZO压缩格式．它们相比GZIP，占用的CPU更少．
- 在大多数情况下，Snappy/LZO的选择都更好.
- Snappy比LZO更好．

## 推荐文章

这里推荐两篇关于不同Block Encoding Type以及压缩算法对磁盘以及性能有什么影响的文章．
- [HBase - Compression vs Block Encoding](http://hadoop-hbase.blogspot.com/2016/02/hbase-compression-vs-blockencoding_17.html)
- [The Effect of ColumnFamily, RowKey and KeyValue Design on HFile Size](https://blogs.apache.org/hbase/entry/the_effect_of_columnfamily_rowkey)
- [HBase最佳实践－列族设计优化](https://blog.csdn.net/javastart/article/details/51820212)
