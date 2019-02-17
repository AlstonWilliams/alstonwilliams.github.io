---
layout: post
title: ZooKeeper源码解析(4)-TxnLog文件格式
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- ZooKeeper源码解析
tags:
- ZooKeeper源码解析
---
在上篇文章中，我们介绍了ZooKeeper中Snapshot文件的格式．在这盘文章中，我们将会介绍TxnLog文件的格式．

在ZooKeeper中，Transaction代表一次客户端操作，而TxnLog文件就保存了这些Transaction．从ZooKeeper中的**FileTxnSnapLog**类中，我们可以看到其格式如下：

~~~~
FileHeader TxnList ZeroPad

FileHeader:{
  magic 4bytes(固定位ZKLG)
  version 4bytes
  dbid 8bytes
}

TxnList:
  Txn || Txn TxnList

Txn:
  checksum Txnlen TxnHeader Record 0x42

checksum: 8bytes 
  Adler32 is currently used, calculated across payload -- Txnlen, TxnHeader, Record and 0x42

Txlen:
  len 4bytes

TxnHeader: {
  clientId 8bytes
  cxid 4bytes
  zxid 8bytes
  time 8bytes
  type 4bytes
}

Record: See Jute definition file for detail on the various record type

ZeroPad: 0 padded to EOF(filled during preallocation stage)
~~~~

TxnLog的文件名是**log.current_zxid**的格式，其中**current_zxid**是创建它的时候的zxid，zxid大于**current_zxid**的Transaction都是保存在这个文件中．默认情况下，这个文件是64MB大小．
