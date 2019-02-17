---
layout: post
title: Hadoop中，Mapper和Reducer究竟背着我们做了什么？
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
在这篇文章中，我们会探究，Mapper和Reducer的一些不为人知的秘密。

为什么说不为人知呢？毕竟Hadoop是开源的，你可以阅读源码获取一切你想要的信息。你要是这样做，我无法反驳，因为这确实是最权威的方式。

在阅读《Hadoop: The Definitive Guide 4th Edition》时，我们都见过这么一副图片，它简单的解释了Mapper和Reducer究竟是如何沟通的，究竟做了些什么：

![](http://upload-images.jianshu.io/upload_images/4108852-934a0f30e20b706f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

尽管这幅图片确实非常正确，但是，很遗憾的是，它并不是非常详细。

至少在我阅读这本书的时候，当时看到这幅图片，只能大致了解一下过程，而对一些细节并不是非常清楚。只有在后来搞明白了，再回来看这幅图片，才觉得确实非常正确。

所以，在这篇文章中，我们会用具体的例子，来一步步地解析这个过程。

但是，有一点需要各位注意的是，我只是用例子一步步地验证了这个过程，而并不是阅读源码来得出了这个过程。尽管实践是检验真理的唯一标准，但是在并不清楚真理是什么而靠实践得到的结果来总结真理的情况下，并不是那么严格，可能会有一些偏差。

所以，各位要做好一个心理准备，就是这篇文章中的内容，可能有错误，各位要有一个质疑的思想。

最近我也是打算阅读Hadoop的源码的，到时也会通过源码来验证这个过程是否正确。

那我们开始吧。

## 总体轮廓

尽管上面我们已经给了一副图片，但是我还是总结了一副我认为更加直观的图片，呈现给大家。

![](http://upload-images.jianshu.io/upload_images/4108852-2e5bc384ed126137.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面我们就会一一介绍为何他们他们是这个顺序。

## 问题

我们会用两个问题来阐述，如果搞懂这两个问题的处理过程，这个过程也就懂了。

问题一：单词计数器。统计有相同首字母的单词中，全部的单词的个数，以及不重复的单词的个数。比如，有**(a, aa, aa, bb, ba)**,那么输出应该是**(a, 3, 2), (b, 2, 2)**。其中每个元组中的第二列为全部单词的数量，第三列为不重复单词的数量。

问题二：单词计数器。跟上面的问题大体相同，只是现在是统计有相同尾字母的单词中，全部的单词的个数，以及不重复的单词的个数。还是上面的那个例子，现在输出应该是**(a, 4, 3), (b, 1, 1)**。

## 过程

#### 问题一

我们先来解决第一个问题。

很显然，我们要是把单词按照首字母分组，那么就跟传统的单词计数器类似了。

所以，我们写一个Partitioner，将单词们按照首字母分区，保证首字母相同的单词会被分到同一个Reducer中。

Partitioner的实现跟**HashPartitioner**的实现类似，如下所示:
![](http://upload-images.jianshu.io/upload_images/4108852-d93bc30ac93b1334.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很简单对吧。

然后我们的Reducer端处理过程如下：

![](http://upload-images.jianshu.io/upload_images/4108852-a2d6d6afd00ef4f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你可能很奇怪，为什么我们明明可以用HashSet来统计不重复单词，而偏偏要采用这种形式。

因为如果采用HashSet，就不会理解它排序的过程。

我们可以看到，我们在Reducer端，就是判断一下是否前一个分区已经处理完毕，如果已经处理完，那么就将结果写入到输出中。最后在cleanup()函数中，再写一遍，防止最后一个分区的结果被漏掉。

那么我们为什么可以这样处理呢？

因为我们的mapper的输出如下:
**(a, 1), (aa, 1), (aa, 1), (bb, 1), (ba, 1)**

然后经过Partitioner之后，产生了这么两个Partition:
- Partition 1: (a, 1), (aa, 1), (aa, 1)
- Partition 2: (bb, 1), (ba, 1)

然后分别两个Reducer来处理他们。当然，如果你强制只有一个Reducer，那么它们还是会被同一个Reducer处理。所以才有了我们在Reducer中判断上一个Partition是否处理完毕的逻辑。

数据在Reducer端会先进行一个排序，那么它是如何进行排序的呢？

默认情况下，是按照Key以及其类型进行排序。这里我们的Key的类型为Text，所以会将这个Reducer接收到的数据按照**Text**类型的**Comparator**进行排序。

我们这里假设仅有一个Reducer。

由于**Text**的**Comparator**会将输入排序成**(a, 1), (aa, 1), (aa, 1), (ba, 1), (bb, 1)**这种顺序，所以上面的代码没有什么问题。

而你需要注意的是，如果你用的是自定义的类型，或者自定义的Comparator，那么经过排序后，可能是乱序的，可能以**a**开头的单词和以**b**开头的单词就是乱序的了。具体的例子，我们会在下一个问题中介绍到。

然后，Reducer端会进行merge操作，会将**(a, 1), (aa, 1), (aa, 1), (ba, 1), (bb, 1)**merge为**((a, 1), (aa, (1, 1)), (ba, 1), (bb, 1))**。

这样看，我们的Reducer是正确的。

但是这里同样有一个坑。

你怎么知道它是如何进行merge的？

这个问题，我们也会在问题二中介绍。

现在你只需要清楚，它也是按照Text进行merge的就好了。

Ok。那结果自然是正确的。

#### 问题二

问题二看起来好像跟问题一类似？

对的。

跟问题一的区别是，我们需要对尾字母进行分区？

Yes。

所以其实有一个很简单的解决这个问题的方式，即把尾字母提到前面去，成为首字母，然后这个问题就跟问题一一样了，类似于动态规划。对吧？

但是这里我们不采用这种方法。因为这种方法对我们理解在问题一中提到的，Reducer端如何进行排序，如何进行merge并没有帮助。

我们采用另一种方法。

还是对尾字母进行排序。

然后我们的Partitioner如下：

![](http://upload-images.jianshu.io/upload_images/4108852-78bc5652b78f1e1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Reducer如下:

![](http://upload-images.jianshu.io/upload_images/4108852-ab1a2b2fdc8c7e47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你会发现，当你的Reducer的数量大于或者等于你的Partition的数量时，everything works well。但是一旦你只有一个Reducer，就出现问题了。

为什么呢？

我们上面提到过，Reducer端会将收到的数据先按照Key以及Key的对应的类型的Comparator进行排序。

假设它收到了这么两个Partition的数据:
- Partition 1: (aa, 1), (ba, 1),
- Partition 2: (bb, 1), (ab, 1)

我们希望它如何排序？

**(aa, 1), (ba, 1), (ab, 1), (bb, 1)**

反正就是相同尾字母的单词都是相邻的。

而实际上，它会按照**Text**的**Comparator**进行排序，那么排序的结果是什么呢？

**(aa, 1), (ab, 1), (ba, 1), (bb, 1)**

那结果自然是不正确。

为什么你的Reducer的数量大于或者等于你的Partition的数量时，Everything works well呢？我想你应该已经有答案了。

好，找到问题所在了。那我们自然就想到解决方案，我们自己定义一个排序函数不就好了？

Yes.

所以我们自定义了一个数据类型，以及一个对应的Comparator。其实我这里写的有些麻烦了，不用自定义数据类型的。各位自己写的时候要注意。

![](http://upload-images.jianshu.io/upload_images/4108852-7573ddc02ba51351.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，在**SuffixCompareText**的**compare(SuffixCompareText o)**中，会先根据尾字母进行排序，如果尾字母相同，则根据剩下的字符串进行排序。

这样我们不仅能保证具有相同尾字母的单词是连续的，还能保证具有相同尾字母且相同的单词也是连续的。

为什么需要这种保证呢？

因为merge。

现在是时候介绍一下merge的过程了。
- 首先查找你是否通过**Job.setGroupComparatorClass(YourComparatorClass)**方法指定了merge的方法，如果有，则根据这个方法，对输入的相邻的数据进行比较，如果相同，则合并。注意，是相邻的数据。比如，如果输入数据是**(aa, 1), (aa, 1), (ab, 1)**，则会被合并为**(aa, (1, 1)), (ab, 1)**。而如果输入数据是**(aa, 1), (ab, 1), (aa, 1)**，则并不会进行合并，结果还是**(aa, 1), (ab, 1), (aa, 1)**。
- 如果你没有指定这个方法，那么就查看你是否通过**Job.setSortComparatorClass(SortComparatorClass)**指定了对数据排序的方法，如果指定了的话，那么会根据这个方法，对输入的相邻的数据进行比较，如果相同，则合并。所以，你可以看到，如果上面的**SuffixCompareText**的**compare**中不注明当尾字母相同时，对剩下的字符串进行比较。那么，**(ab, 1), (cb, 1), (bb, 1)**会被合并为**(ab, (1, 1, 1))**。因为在**compare()**方法中，它们比较的结果为0，即相同。这样我们就无法统计不重复单词的数量了。
- 如果你上面的两个方法都没有指定，那么就根据你的Reducer的input的Key进行合并。也是根据Key的**Comparator**。

其实不管怎么说，由于merge时，是对相邻的数据进行对比的，所以你一定需要让Reducer拿到的数据是有序的。

这里似乎GroupComparator没有什么用，因为我们的输入的顺序已经是有序的了。但是，理解它的过程，也是有一些好处的。比如，万一以后你不仅仅只是想按照相同的key进行merge呢？

## 总结

在上文中，我们已经解密了mapper到reducer的过程。理解这些对你以后会大有裨益的。

## 源码

关于这个Demo的源码，我已经发布到Github上了，点击[这里](https://github.com/AlstonWilliams/WordCountForHadoop)。

你也可以复制这个链接到浏览器来查看。
https://github.com/AlstonWilliams/WordCountForHadoop
