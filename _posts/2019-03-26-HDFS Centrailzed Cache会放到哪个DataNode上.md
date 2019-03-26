---
layout: post
title: HDFS Centrailized Cache会放到哪个DataNode上
date: 2019-03-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---

前几天，在看Hadoop User Email List的时候，发现了一个关于HDFS Centrailzed Cache的问题。刚好我又不熟悉这块，甚至之前都没听说过，就好好了解了一番。

其实原理很简单，各位读一下下面的几个链接，就能清楚是怎么回事:
- [Centralized Cache Management in HDFS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/CentralizedCacheManagement.html)
- [HDFS 集中式的缓存管理原理与代码剖析](https://infoq.cn/article/hdfs-centralized-cache)
- [In-memory Caching in HDFS: Lower Latency, Same Great Taste](https://www.slideshare.net/Hadoop_Summit/inmemory-caching-in-hdfs-lower-latency-same-great-taste-33921794)

其实上面第二篇文章中已经介绍了，会将Cache放在这个 block 的三个 replica 所在的 DataNode 其中的剩余可用内存最多的一个上。但是当时我没细看，就自己阅读了一下源码来探究这个问题。

相关代码主要在`CacheReplicationMonitor.chooseDatanodesForCaching`中:
~~~
  /**
   * Chooses datanode locations for caching from a list of valid possibilities.
   * Non-stale nodes are chosen before stale nodes.
   *
   * @param possibilities List of candidate datanodes
   * @param neededCached Number of replicas needed
   * @param staleInterval Age of a stale datanode
   * @return A list of chosen datanodes
   */
  private static List<DatanodeDescriptor> chooseDatanodesForCaching(
      final List<DatanodeDescriptor> possibilities, final int neededCached,
      final long staleInterval) {
    // Make a copy that we can modify
    List<DatanodeDescriptor> targets =
        new ArrayList<DatanodeDescriptor>(possibilities);
    // Selected targets
    List<DatanodeDescriptor> chosen = new LinkedList<DatanodeDescriptor>();

    // Filter out stale datanodes
    List<DatanodeDescriptor> stale = new LinkedList<DatanodeDescriptor>();
    Iterator<DatanodeDescriptor> it = targets.iterator();
    while (it.hasNext()) {
      DatanodeDescriptor d = it.next();
      if (d.isStale(staleInterval)) {
        it.remove();
        stale.add(d);
      }
    }
    // Select targets
    while (chosen.size() < neededCached) {
      // Try to use stale nodes if we're out of non-stale nodes, else we're done
      if (targets.isEmpty()) {
        if (!stale.isEmpty()) {
          targets = stale;
        } else {
          break;
        }
      }
      // Select a random target
      DatanodeDescriptor target =
          chooseRandomDatanodeByRemainingCapacity(targets);
      chosen.add(target);
      targets.remove(target);
    }
    return chosen;
  }
~~~

以及`CacheReplicationMonitor.addNewPendingCached`中:
~~~
 /**
  * Add new entries to the PendingCached list.
  *
  * @param neededCached     The number of replicas that need to be cached.
  * @param cachedBlock      The block which needs to be cached.
  * @param cached           A list of DataNodes currently caching the block.
  * @param pendingCached    A list of DataNodes that will soon cache the
  *                         block.
  */
 private void addNewPendingCached(final int neededCached,
     CachedBlock cachedBlock, List<DatanodeDescriptor> cached,
     List<DatanodeDescriptor> pendingCached) {
   // To figure out which replicas can be cached, we consult the
   // blocksMap.  We don't want to try to cache a corrupt replica, though.
   BlockInfoContiguous blockInfo = blockManager.
         getStoredBlock(new Block(cachedBlock.getBlockId()));
   if (blockInfo == null) {
     LOG.debug("Block {}: can't add new cached replicas," +
         " because there is no record of this block " +
         "on the NameNode.", cachedBlock.getBlockId());
     return;
   }
   if (!blockInfo.isComplete()) {
     LOG.debug("Block {}: can't cache this block, because it is not yet"
         + " complete.", cachedBlock.getBlockId());
     return;
   }
   // Filter the list of replicas to only the valid targets
   List<DatanodeDescriptor> possibilities =
       new LinkedList<DatanodeDescriptor>();
   int numReplicas = blockInfo.getCapacity();
   Collection<DatanodeDescriptor> corrupt =
       blockManager.getCorruptReplicas(blockInfo);
   int outOfCapacity = 0;
   for (int i = 0; i < numReplicas; i++) {
     DatanodeDescriptor datanode = blockInfo.getDatanode(i);
     if (datanode == null) {
       continue;
     }
     if (datanode.isDecommissioned() || datanode.isDecommissionInProgress()) {
       continue;
     }
     if (corrupt != null && corrupt.contains(datanode)) {
       continue;
     }
     if (pendingCached.contains(datanode) || cached.contains(datanode)) {
       continue;
     }
     long pendingBytes = 0;
     // Subtract pending cached blocks from effective capacity
     Iterator<CachedBlock> it = datanode.getPendingCached().iterator();
     while (it.hasNext()) {
       CachedBlock cBlock = it.next();
       BlockInfoContiguous info =
           blockManager.getStoredBlock(new Block(cBlock.getBlockId()));
       if (info != null) {
         pendingBytes -= info.getNumBytes();
       }
     }
     it = datanode.getPendingUncached().iterator();
     // Add pending uncached blocks from effective capacity
     while (it.hasNext()) {
       CachedBlock cBlock = it.next();
       BlockInfoContiguous info =
           blockManager.getStoredBlock(new Block(cBlock.getBlockId()));
       if (info != null) {
         pendingBytes += info.getNumBytes();
       }
     }
     long pendingCapacity = pendingBytes + datanode.getCacheRemaining();
     if (pendingCapacity < blockInfo.getNumBytes()) {
       LOG.trace("Block {}: DataNode {} is not a valid possibility " +
           "because the block has size {}, but the DataNode only has {}" +
           "bytes of cache remaining ({} pending bytes, {} already cached.",
           blockInfo.getBlockId(), datanode.getDatanodeUuid(),
           blockInfo.getNumBytes(), pendingCapacity, pendingBytes,
           datanode.getCacheRemaining());
       outOfCapacity++;
       continue;
     }
     possibilities.add(datanode);
   }
   List<DatanodeDescriptor> chosen = chooseDatanodesForCaching(possibilities,
       neededCached, blockManager.getDatanodeManager().getStaleInterval());
   for (DatanodeDescriptor datanode : chosen) {
     LOG.trace("Block {}: added to PENDING_CACHED on DataNode {}",
         blockInfo.getBlockId(), datanode.getDatanodeUuid());
     pendingCached.add(datanode);
     boolean added = datanode.getPendingCached().add(cachedBlock);
     assert added;
   }
   // We were unable to satisfy the requested replication factor
   if (neededCached > chosen.size()) {
     LOG.debug("Block {}: we only have {} of {} cached replicas."
             + " {} DataNodes have insufficient cache capacity.",
         blockInfo.getBlockId(),
         (cachedBlock.getReplication() - neededCached + chosen.size()),
         cachedBlock.getReplication(), outOfCapacity
     );
   }
 }
~~~

我们可以看到，逻辑就是，从满足下面这几个条件的DataNode中，选择一个可用Cache内存最多的节点:
1. 此DataNode状态正常，没有decommission
2. 这个Block在这个DataNode上，不是corrupt状态
3. 这个Block没有在DataNode上面缓存过(pendingCached以及cached中都没有此DataNode)
4. 这个Block已经关闭(比如当一个Block在进行replication的时候，如果第二个replicate 2 和replication 3没有完成，那么就不能选择这些DataNode)

那策略介绍完了。这里我就有一个新的疑问了，我们知道，MapReduce的Mapper任务，会尽量被分配到有相应Block的那个节点上。那这儿会考虑Centrailzed Cache么? 对Spark等其它框架呢?

关于Mapper任务分配时是否会考虑Centrailzed Cache，我会查看相关源码，并整理上来。
