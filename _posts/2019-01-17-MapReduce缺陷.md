---
layout: post
title: MapReduce缺陷
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
作为Hadoop的一个核心，MapReduce一直都是人们讨论的热点．而且，各类书籍上，往往也只是介绍了MapReduce的优点，其执行过程．对其缺陷，却并没有一个清晰的说明．包括你在百度，在Google上面用中文搜索"MapReduce"的局限性，都是得不到有价值的结果的．倒是有很多论文，专门讨论这个问题．

作为一项技术，缺陷肯定是有的．这不，今天在尝试比较深入的使用它的时候，就碰到了几个坑．之前只是尝试Tutorial中的WordCount那种例子，一直也没有认识到它的局限性．

我这里讨论的是，我们编程人员在直接使用MapReduce这种框架来编程的情况下，MapReduce的局限性．Hive,Pig等同样都是基于MapReduce,不也很方便吗？


## 缺陷
- 如果需要进行复杂的计算，则需要流式的串行计算．MapReduce在运行的时候，是并行计算的，比如Map阶段和Reduce阶段．然而，MapReduce本身的这两个阶段，大多数情况下，是完成不了一次稍微复杂一些的运算的．比如说，我有一个日志文件，其中有用户的IP，访问的时间，以及访问的URL.如果我们想要计算用户的访问次数，并按值的顺序或者倒序来进行排列．是很费事的．我们需要现在一个MapReduce的Job中，统计出来用户的访问次数．在另一个MapReduce的Job中，将其按值的顺序排列．所以后面的那个MapReduce的Job，需要等待前面那个完成．当然，我们也可以实现一个Comporable接口，进行比较，这样就不需要实现两个Job.但是，这样程序的复杂度就提高了．
- 开发时，调试起来困难．据说使用IDE的情况下，也是蛮轻松的．但是，我写MapReduce时，都是用的VIM．今天出现了一个问题，就是MapReduce跑完了，中间没有报任何错误，但是查看结果时，发现什么结果都没有．我尝试使用**System.out.println**语句输出，来定位一下问题．然而，这些语句都不起作用．Google了一下，发现是在运行这个Job时，由于是使用的**hadoop -jar**命令来运行的，所以任务直接是在Hadoop集群中跑的．Hadoop将这些语句的输出，都收集起来了，作为自己的输出，要看的话，需要到JobTracker的WebUI中，查看日志．StackOverflow上有人说，搭建一个本地的单实例模式，然后用正常运行Java程序的方式来运行，就能看到我们使用**System.out.println**输出的内容．
- 只能进行简单的运算．这个缺陷我们在第一条中就说了．我用那个日志的数据集，想实现一个稍微复杂一些的运算，就得写好几个MapReduce.
- 输出结果的形式单一．Reduce阶段的输出，就是一个<Key, Value>的键值对，然而，很多时候，我们的输出结果不是这么简单的．
- 编程模型复杂．尽管MapReduce已经简化了我们的编程模型．但是，不可否认的是，还是很复杂．特别是一个任务需要使用多个MapReduce的Job来实现时．所以，很少有人直接就裸跑MapReduce.而是使用Hive,Pig等．
- 不能写入到已经存在内容的目录．这实际上不算是MapReduce的缺陷，它是HDFS的一个特点．HDFS的特色就是修改数据不方便．所以，我在写MapReduce的程序的时候，不得不写一个脚本来负责清理输出目录，重新编译程序以及打包的操作．

这是目前我在尝试用MapReduce进行数据分析时，踩到的地雷．我们看一下Quora上有人对MapReduce的局限性的回答:

![1.png](http://upload-images.jianshu.io/upload_images/4108852-644b4edc40a65f06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
