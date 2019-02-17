---
layout: post
title: Hive中的Managed-Table以及External-Table(译)
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 大数据
tags:
- 大数据
---
> 译者注:
 这篇文章中主要介绍了Hive中的Managed Table和External Table在存储文件上的区别．更多的区别，请自行搜索．

 在这篇文章中，我们主要介绍题目中所说的那两种Table的区别，以及怎样创建这两种类型的表和什么时候我们会用到它们．

 Hive有两种类型的表:
 - Managed Table
 - External Table

下面我们详细介绍这两种表．

## Managed Table
这种表也被称作Internal Table.这是Hive中的默认的类型．如果你在创建表的时候没有指明Managed或者External,那么默认就会给你创建Managed Table.

Managed Table的数据，会存放在HDFS中的特定的位置中，通常是*/user/hduser/hive/warehouse*.当然，也不一定，看你的Hive的配置文件中是如何配置的．

我们可以使用下面的命令来创建一个Managed Table:

![](https://i2.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/Create_manage_table.png?resize=768%2C59&ssl=1)

我们已经成功创建一个Managed Table了，我们使用下面的命令来查看:

![](https://i2.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/info_managed_table.png?resize=763%2C577&ssl=1)

在上图中，我们可以看到，*Table Type*那里是*Managed Table*,这就表示我们确实创建了一个Managed类型的Table.

我们使用下面的命令，从本地文件系统中加载一些数据到这张表中:

![](https://s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/load_data_managed_table.png)

通过下面的命令，查看这张表在HDFS中的位置:

![](https://i1.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/check_managed_data.png?resize=768%2C64&ssl=1)

我们可以从上图看到该表在HDFS中的位置．

下面我们删除这张表:

![](https://i2.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/drop_managed_table.png?resize=245%2C74&ssl=1)

我们成功地删除了这张表．

我们再查看一下这种表在HDFS中是否存在:

![](https://i0.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/check_data_after_delete_managed.png?resize=768%2C55&ssl=1)

在上图中，我们可以看到，表和它的内容，都从HDFS中删除了．

## External Table
External Table特别适用于想要在Hive之外使用表的数据的情况．当你删除External Table时，只是删除了表的元数据，它的数据并没有被删除．

我们创建一张External Table,并查看一下它的结构:

![](https://i2.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/create_external_table.png?resize=768%2C522&ssl=1)

从上图中，我们可以看到，表的类型是"External Table".

现在我们同样从文件系统中加载一些数据到这张表中:

![](https://s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/load_data_external_table.png)

然后，我们可以通过下面的命令查看它的位置:

![](https://i1.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/check_managed_data.png?resize=768%2C64&ssl=1)

删除它:

![](https://i2.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/drop_external_table.png?resize=245%2C74&ssl=1)

再检查一下:

![](https://i0.wp.com/s3.amazonaws.com/acadgildsite/wordpress_images/bigdatadeveloper/Managed+and+External+tables+in+hive/check-data_after_delete_external.png?resize=768%2C72&ssl=1)

你可以看到，即使我们删除了表，它的数据仍然存在．

## 什么时候使用哪种表?

### Managed Table
- 数据是临时数据
- 外部的程序无法访问这些数据
- 数据会随着表的删除而删除

### External Table
- 数据可以被外部程序访问
- 你不能基于已经存在的表再创建表
- 表被删除时，数据不会被删除
