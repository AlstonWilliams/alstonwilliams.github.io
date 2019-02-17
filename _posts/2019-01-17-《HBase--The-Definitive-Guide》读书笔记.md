---
layout: post
title: 《HBase--The-Definitive-Guide》读书笔记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
## HBase

 All rows are always sorted lexicographically by their row key.

The row keys can be any arbitrary array of bytes and are not necessary human-readable.

Rows are composed of columns, and those, in turn, are grouped into column families.

All columns in a column family are stored together in the same low-level storage file, called an HFile.

Column families need to be defined when the table is created and should not be changed too often, nor should there be too many of them. There are a few known short comings in the current implementation that force the count to be limited to the low tens, but in practice it is often a much smaller number. The name of the column family must be composed of printable characters, a notable difference from all other names or values.

Columns are often referenced as `family:qualifier` with the `qualifier` being any arbitrary array of bytes.

Every column value, or cell, either is timestamped implicitly by the system or can be set explicitly by the user. This can be used, for example, to save multiple versions of a value as it changes over time.

The user can specify how many versions of a value should be kept. In addition, there is support for predicate deletions allowing you to keep, for example, only values written in the past week.

~~~
(Table, Rowkey, Family, Column, TimeStamp) -> Value
SortedMap<RowKey, List<SortedMap<Column, List<Value, Timestamp>>>>
~~~

## Auto-sharding

The basic unit of scalability and load balancing in HBase is called a `region`. Regions are essentially contiguous range of rows stored together. They are dynamically split by the system when they become too large. Alternatively, they may also be merged to reduce their number and required storage files.

Initially there is only one region for a table, and as you start adding data to it, the system is monitoring it to ensure that you do not exceed a configured maximum size. If you exceed the limit, the region is split into two at the middle key-the row key in the middle of the region-creating two roughly equal halves.

Each region is served by exactly one region server, and each of these servers can serve many regions at any time.

## HFile

The data is stored in store files, called HFiles, which are persistent and ordered immutable maps from keys to values. Internally, the files are sequences of blocks with a block index stored at the end. The index is loaded when the HFile is opened and kept in memory. The defualt block size is 64KB but can be configured differently if required.

The store files are typically saved in the Hadoop Distribtued File System.

When data is updated it is first written to a commit log, called a `write-ahead log` in HBase, and then stored in the in-memory memstore. Once the data in memory has exceeded a given maximum value, it is flushed as an HFile to disk. After the flush, the commit logs can be discarded up to the last unflushed modification. While the system is flushing the memstore to disk, it can continue to serve readers and writers without having to block them. This is archieved by rolling the memstore in memory where the new/empty one is taking the updates, while the old/full one is converted into a file. Note that the data in the memstores is already sorted by keys matching exactly what HFile represent on disk, so no sorting or other special processing has to be performed.

Because store files are immutable, you cannot simply delete values by removing the key/value pair from them. Instead, a `delete marker` is written to indicate the fact that the given key has been deleted. During the retrival process, these delete markers mask out the actual values and hide them from reading clients.

Reading data back involves a merge of what is stored in the memstores, that is, the data that has not been written to disk, and the on-disk store files. Note that the WAL is never used during data retrieval, but solely for recovery purposes when a server has crashed before writing the in-memory data to disk.

Since flushing memsotres to disk causes more and more HFiles to be created, HBase has a housekeeping mechanism that merges the file into larger ones using compaction. There are two types of compaction: minor compactions and major compactions. The former reduce the number of storage files by rewritting smaller files into fewer but larger ones, performing an n-way merge. Since all the data is already sorted in each HFile, that merge is fast and bound only by disk I/O performance.

The major compactions rewrite all files within a column family for a region into a single new one. They also have another distinct feature compared to the minor compactions: based on the fact that they scan all key/value pairs, they can drop deleted entries including their deletion marker.

The master server is also responsible for handling load balancing of regions across region servers, to unload busy servers and more regions to less occupied ones. The master is not part of the actual data storage or retrieval path. If negotiates load balancing and maintains the state of the cluster, but never provides any data services to either the region servers or the clients, and is therefore lightly loaded in practice. In addition, it takes care of schema changes and other metadata operations, such as creation of tables and column families.

 Region servers are responsible for all read and write requests for all regions they serve, and also split regions that have exceeded the configured region size thresholds. Client communicate directly with them to handle all data-related operations.

![](https://upload-images.jianshu.io/upload_images/4108852-15f981c272f68aef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

You can estimate the number of required file handles roughly as follows. Per column family, there is at least one storage file, and possibly up to five or six if a region is under load; On average, though, there are three storage files per column family. To determine the number of required file handles, you multiply the number of column families by the number of regions per region server. For example, say you have a schema of 3 column families per region and you have 100 regions per region server. The JVM will open 3\*3\*100 storage files=900 file descriptors, not counting open JAR files, configuration files, CRC32 files, and so on.

Run `lsof -p REGIONSERVER_PID` to see the accurate number.

![](https://upload-images.jianshu.io/upload_images/4108852-08f0fb746493c291.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## HBase client API

All operations that mutate data are guaranteed to be atomic on a per-row basis.

Creating `HTable` instances is not without cost. Each instantiation involuves scaning the .META.table to check if the table actually exists and if it is enabled, as well as a few other operations that make this call quite costly. Therefore, it is recommended that you create HTable instances only once-and once per thread - and reuse that instance for the rest of the lifetime of your client application.

Be aware that you should not mix a `Delete` and `Put` operation for the same row in one batch call. The operations will be appliedin a different order that guarantees the best performance, but also causes unpredictable results. In same cases, you may see fluctuating results to rece conditions.

Possible result values returned by the batch() calls
![](https://upload-images.jianshu.io/upload_images/4108852-b2cff8764a7ffdf5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

When you use the batch() functionality, the included `Put` instances will not be buffered using the client-side write buffer. The batch() calls are synchronous and send the operations directly to the servers; no delay or other intermediate processing is used. This is obviously compared to the `put()` call, so choose which one you want to use carefully.

**void batch(List<Row> actions, Object[] results)** gives you access to the partial results, while **Object[] batch(List<Row> actions)** does not.

Each unique lock, provided by the server for you, or handed in by you through the client API, guards the row it pertains to against any other lock that attempts to access the same row. In other words, locks must be taken out against an entire row, specifying its row key, and-once it has been acquired-will protect it against any other concurrent modification.

Each call to `Scanner.next()` will be a separate RPC for each row.

## Filter

The possible comparison operators for CompareFilter-based filters

![](https://upload-images.jianshu.io/upload_images/4108852-0b2cb7d12bbac27f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The HBase-supplied comparators, used with CompareFilter-based Filters

![](https://upload-images.jianshu.io/upload_images/4108852-c7b4d9b4c8052260.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Comparison Filters

~~~
compareFilter(CompareOp valueCompareOp, WritableByteArrayComparable valueComparator)
~~~

The general contract of the HBase filter API means you are filtering out information-filtered data is omitted from the results returned to the client. The filter is not specifying what you want to have, but rather what you do not want to have returned when reading data.

In contrast, all filters based on CompareFilter are doing the opposite, in that they include the matching values. In other words, be careful when choosing the comparison operator, as it makes the difference in regard to what the server returns. For example, instead of using LESS to skip some information, you may need to use GREATER_OR_EQUAL to include the desired data points.

- RowFilter
This filter gives you the ability to filter data based on row keys.
- FamilyFilter
This filter works very similar to the RowFilter, but applies the comparison to the column families available in a row - as apposed to the row key.
- Qualifier Filter
This filter works very similar to the RowFilter, but applies the comparision to the column qualifier available in a row.
- ValueFilter
This filter makes it possible to include only columns that have a specific value.
- DependentColumnFilter
It lets you specify a `dependent` column - or `reference` column - that controls how other columns are filtered. It uses the timestamp of the reference column and includes all other columns that have the same timestamp. Here are the constructors provided:
**DependentColumnFilter(byte[] family, byte[] qualifier)
DependentColumnFilter(byte[] family, byte[] qualifier, boolean dropDependentColumn)
DependentColumnFilter(byte[] family, byte[] qualifier, boolean dropDependentColumn, CompareOp valueCompareOp, WritableByteArrayComparable valueComparator)**
Think of it as a combination of a `ValueFilter` and a filter selecting on a reference timestamp.
- DedicatedFilters
  - SingleColumnValueFilter

  You can use this filter when you exactly one column that decides if an entire row should be returned or not. You need to first specify the column you want to track, and then some value to check against. The constructor offered are:
**SingleColumnValueFilter(byte[] family, byte[] qualifier, CompareOp compareOp, byte[] value)
SingleColumnValueFilter(byte[] family, byte[] qualifier, CompareOp compareOp, WritableByteArrayComparable comparator)**

  - SingleColumn
    - PrefixFilter
    Given a `prefix`, specified when you instantiate the filter instance, all rows that match this prefix are returned to the client. The constructor is:
    **public PrefixFilter(byte[] prefix)**
    Compare by row key
    - PageFilter
    You paginate though rows by employing this filter. When you create the instance, you specify a `pageSize` parameter, which controls how many rows per page should be returned.
    - KeyOnlyFilter
    Some applications need to access just the keys of each keyValue, while omitting the actual data. The `KeyOnlyFilter` provides this functionality by applying the filter's ability to modify the processed columns and cells, as they pass through.
    - FirstKeyOnlyFilter
    - InclusiveStopFilter
    The row boundaries of a scan are inclusive for the start row, yet exclusive for the stop row. You can overcome the stop row semantics using this filter, which includes the specified stop row.
    - TimestampsFilter
    When you need fine-grained control over what versions are included in the scan result, this filter provides the means.
    - ColumnCountGetFilter
    You can use this filter to only retrive a specific maximum number of columns per row.
    - ColumnPaginationFilter
    Similar to the `PageFilter`, this one can be used to page through columns in a row. Its constructor has two parameters:
    **ColumnPaginationFilter(int limit, int offset)**
    It skips all columns up to the number given as `offset`, and then includes `limit` columns afterward.
    - ColumnPrefixFilter
    Analog to the `PrefixFilter`, which worked by filtering on row key prefixes, this filter does the same for columns. You specify a prefix when creating the filter:
    **ColumnPrefixFilter(byte[] prefix)**
    All columns that have the given prefix are then included in the result.
    - RandomRowFilter
    There is a filter including random rows into the result. The constructor is given a parameter named `change`, which represents a value between 0.0 and 1.0:
    **RandomRowFilter(float chance)**
- DecoratingFilters
  - SkipFilter
  This filter wraps a given filter and extends it to exclude an entire row, when the wrapped filter hints for a `keyValue` to be skipped.
  The wrapped filter must implement the `filterKeyValue()` method, or the SkipFilter will not work as expected.
  - WhileMatchFilter
  Abort the entire scan once a piece of information is filtered.

## FilterList

Have more than one filter being applied to reduce the data returned to your client application.

~~~
FilterList(List<Filter> rowFilters)
FilterList(Operator operator)
FilterList(Operator operator, List<Filter> rowFilters)
~~~

Operator enumeration:
![](https://upload-images.jianshu.io/upload_images/4108852-b3aafb8a39f6589e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Customize Filter
- filterRowKey(byte[] buffer, int offset, int length)
The next check is against the row key, using this method of the `Filter` implemention. You can use it to skip an entire row from being further processed. The RowFilter uses it to suppress entire rows being returned to the client.
- filterKeyValue(KeyValue v)
When a row is not filtered(yet), the framework proceeds to invoke this method for every `KeyValue` that is part of the current row. The `ReturnCode` indicates what should happen with the current value.
- filterRow(List<KeyValue> kvs)
Once all row and value checks have been performed, this method of the filter is called, giving you access to the list of keyValue instances that have included by the previous filter methods. The `DependentColumnFilter` uses it to drop those columns that do not match the reference column.
- filterRow()
After everything else was checked and invoked, the final inspection is performed using filterRow(). A filter that uses this functionality is the `PageFilter`, checking if the number of rows to be returned for one iteration in the pagination process is reached, returning true afterward. The default `false` would include the current row in the result.
- reset()
This resets the filter for every new row and scan is iterating over. It is called by the server, after a row is read, implicity. This applies to `get` and `scan` operations, alghough obviously it has not effect for the former, as `get` only read a single row.
- filterAllRemaing()
This method can be used to stop the scan, by returning true. It is used by filters to provide the `early out` optimizations mentioned earlier. If a filter returns false, the scan is continued, and the aforementioned methods are called.

![](https://upload-images.jianshu.io/upload_images/4108852-b415531c14e1234f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Counters

Counters make us treat columns as counters.

The increment value and its effect on coutner increments:
![](https://upload-images.jianshu.io/upload_images/4108852-eb0865b4f64268dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Coprocessors

Earlier we discussed how you can use filters to reduce the amount of data being sent over the network from the servers to client. With the coprocessor feature in HBase, you can even move part of the computation to where the data lives.

A coprocessor enables you to run arbitrary code directly on each region server. More precisely, it executes the code on a per-region basis, giving you trigger-like functionality similar to stored procedures in the RDBMS world. From the client side, you do not have to take specific actions, as the framework handles the distributed nature transparently.

They fall into two main groups: Observer and endpoint.
- Observer
This type of coprocessor is comparable to `triggers`: callback functions are executed when certain events occur. This includes user-generated, but also server-internal, automated events.
Observers provide you with well-defined event callbacks, for every operation a cluster server may handle.
The interfaces provided by the coprocessor framework are:
  - RegionObserver
  You can handle data manipulation events with this kind of observer. They are closely bound to the regions of a table.
  - MasterObserver
  This can be used to react to administrative or DDL-type operations. These are cluster-wide events.
  - WALObserver
  This provides hook into the write-ahead log processing
- Endpoint
  Endpoints are dynamic extensions to the RPC protocol, adding callable remote procedures. Think of them as stored procedures, as known from RDBMS. They may be combined with observer implementations to directly interact with the server-side state.

The provided `CoprocessorEnvironment` instance is used to retain the state across the lifespan of the coprocessor instance.

`CoprocessorHost` class that maintains all the coprocessor instances and therir dedicated environment. There are specific subclasses, depending on where the host is used, in other words, on the master, region server, and so on.

The trinity of `Coprocessor`, `CoprocessorEnvironment`, and `CoprocessorHost` forms the basic for the classes that implement the advanced functionality of HBase, depending on where they are used. They provide the life-cycle support for the coprocessors, manage their state, and offer the environment for them to execute as expected.

![](https://upload-images.jianshu.io/upload_images/4108852-d9eec8449dbe93a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Specifying the column family as part of the query can eliminate the need to search the separate storage files. If you only need the data of one family, it is highly recommended that you specify the family for your read operation.

Finding the right balance between sequential read and write operation:
![](https://upload-images.jianshu.io/upload_images/4108852-9fc177cb6d9932b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Using the salted or promoted field keys can strike a good balance of distribution for write operation, and sequential subsets of keys for read performance. If you are only doing random reads, it makes most sense to use random keys: this will avoid creating region hot-spots.

## Performance Tuning

1. Improve GC
2. Compression
3. Optimizing splits and compactions
  Do split and compact manully
4. Region Hotspotting
  You may need to salt the keys, or use random keys to distribute the load across all servers evenly.
  Sometimes an existing table with many regions is not distributed well - in other words, most of its regions are located on the same region server. This means that, although you insert data with random keys, you still load one region server much more other than the others. You can use the move() function, from the HBase Shell, or use the HBaseAdmin class to explicitly move the server's table regions to other servers. Alternatively, you can use the unassign() method or shell command to simply remove a region of the affected table from the current server. The master will immediately deploy it on another available server.
5. Presplitting Regions
6. Load Balancing
  The master has a built-in feature, called the `balancer`. By default, the balancer runs every five minutes, and it is configured by the `hbase.balancer.period` property.
7. Merging Regions
  HBase ships with a tool that allows you to merge two adjacent regions as long as the cluster is not online.

## Client API: Best Practices

1. Disable auto-flush
When performing a lot of put operations, make sure the auto-flush feature of HTable is set to false, using the setAutoFlush(false) method. Otherwise, the `Put` instances will be sent one at a time to the region server. Puts added via HTable.add(Put) and HTable.add(<List>Put) wind up in the same write buffer. If auto-flushing is disabled, these operations are not sent until the write buffer is filled. To explicitly flush the messages, call flushCommits(). Calling `close` on the HTable instance will implicitly invoke flush commits().

2. Use scanner-caching
If HBase is used as an input source for a MapReduce job, for example, make sure the input `Scan` instance to the `MapReduce` job has `setCaching()` set to something greater than the default of 1. Using the default value means that the map task will make callbacks to the region server for every record processed. Setting this value to 500, for example, will transfer 500 rows at a time to the client to be processed. There is a cost to having the cache value be larger because it costs more in memory for both the client and region server, so bigger is not always better.

3. Limit scan scope
Whenever a Scan is used to process large numbers of rows(and especially when used as a MapReduce source), be aware of which attributes are selected. If `Scan.addFamily()` is called, all of the columns in the specified column family will be returned to the client. If only a small number of the available columns are to be processed, only those should be specified in the input scan because column overselection incurs a nontrivial performance penacty over large data sets.

4. Close `ResultScanners`
This isn't so much about improving performance, but rather avoid performance problems. If you forget to close `ResultScanner` instances, as returned by `HTable.getScanner`, you can cause problems on the region servers.

5. Block cache usage
`Scan` instances can be set to use the block cache in the region server via the setCacheBlocks() method. For scans used with MapReduce jobs, this should be false. For frequently accessed rows, it is advisable to use block cache.

6. Optimal loading of row keys
When performing a table scan where only the row keys are needed(no families, qualifiers, value, or timestamps), add a FilterList with a MUST_PASS_ALL operator to the scanner using setFilter(). The filter list should include both a `FirstKeyOnlyFilter` and a `KeyOnlyFilter` instance. Using this filter combination will cause the region server to only load the row key of the first keyValue found and return it to the client, resulting in minimized network traffic.

7. Turn off WAL on Puts
A frequently discussed option for increasing throughput on Puts is to call `writeToWAL(false)`. Turning this off means that the region server will not write the `Put` to the write-ahead log, but rather only into the memstore. However, the consequence is that if there is a region server failure there will be data loss. If you use `writeToWAL(false)`, do so with extrence caution. You may find that it actually makes little difference if your load is well distributed across the cluster.

## How Client communicates with HBase

A new client contracts the ZooKeeper ensemble first when trying to access  a particular row. It does so by retrieving the server name that hosts the `-ROOT` region from ZooKeeper. With this information it can query that region server to get the server name that hosts the `.META.` table region containing the row key in question. Both of these details are cached and only looked up once. Lastly, it can query the reported .META. server and retrive the server name that has the region containing the row key the client is looking for.

Once it has been told in what region the row resides, it caches this information as well and contacts the HRegionServer hosting that region directly. So, over time, the client has a pretty complete picture of where to get rows without needing to query the .META. server again.

The HMaster is responsible for assigning the regions to each HRegionServer when you start HBase. This also includes the special `-ROOT-` and `.META.`

The HRegionServer opens the region and creates a corresponding HRegion object. When the HRegion is opened it sets up a `store` instance for each `HColumnFamily` for every table as defined by the user beforehand. Each `store` instance can, in turn, have one or move `StoreFile` instances, which are lightweight wrappers around the actual storage file called `HFile`. A `Store` also has a `MemStore`, and the `HRegionServer` a shared HLog instance.

## What happened when issue `HTable.put(Put)`?

THe first step is to write the data to the write-ahead log(the WAL), represented by the HLog class. The `WAL` is a standard Hadoop `SequenceFile` and it store `HLogKey` instances. These keys contain a sequential number as weel as the actual data and are used to replay not-yet-persisted data after a server crash.

Once the data is written to the WAL, it is placed in the `MemStore`. At the same time, it is checked to see it the `MemStore` is full, and if so, a flush to disk is required. The request is served by a separate thread in the HRegionServer, which writes the data to a new `HFile` located in HDFS. It also saves the last written sequence number so that the system knows what was persisted so far.
