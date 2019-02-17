---
layout: post
title: 《Hadoop--The-Definitive-Guide》读书笔记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
 Goal of MapReduce:
- Serve the tasks which needs only several minutes or several hours
- Run in a data center which has high bandwidth
- The machine in the data center is high available

YARN is a resource manager in the cluster, which allows any distributed system(not only MapReduce) run on the Hadoop.

New APIs vs Old APIs:
The new APIs and old APIs are generally provided by same jars, but loactes in different packages. The new APIs are located in package **org.apache.hadoop.mapreduce**, but the old APIs are located in package **org.apache.hadoop.mapred**.

Hadoop will split the job into many tasks to run, includes 'map' and 'reduce'. These tasks run in the cluster, and be managed by YARN. One task will be re-run in different node if failure.

Hadoop will split the input into little data blocks who has same size, which is called 'input slice'. And Hadoop will run 'map' task for every 'slice'. But it is incorrect that more slices will be better.

If slices are too many, the time for managing metadata like building 'map tasks' will too long.

A convenient slice size is like block size in HDFS for almost task, which is 128MB default. Why?

For example, if slice is double block, then when we execute the task, the two blocks may not be inside same machine, so the block will be transferred between different machine to complete the task, and the block may be transferred between different data center even worse.

'Map' will write it's output to local fs, not HDFS. And the 'Map' task will be re-do if the output of 'Map' is failure to be transferred to 'Reduce'. But 'Reduce' will write it's output to HDFS.

If there are many 'reduce', 'map' will partition their output. Partition can be controlled by customized partition method, but generally we use default partition method which is efficiency.

![](http://upload-images.jianshu.io/upload_images/4108852-f0e5d01ad46e5295.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

We can use 'combiner' to reduce the data transferred between 'Map' and 'Reduce'.

Hadoop Streaming:
Hadoop Streaming use Unix Std IO as the interface between Hadoop and application, so we can use any programming language to write MapReduce by Std IO.

HDFS features:
- Can store big file
- Streaming data access
- Commercial hardware
- Don't support operations which used low latency
- Don't use it to store so many little file, it is better to store a single big file. Because file has metadata which also accompany space.
- Just support single one user write, and data is always appended to the end of file.

HDFS concepts:
- Data Block:
  Block in HDFS is 128MB in default. And file with smaller size will not accompany the entire block. For example, a 1MB file will only accompany 1MB but not 128MB.
- NameNode
  1. 命名空间镜像文件
  2. 编辑日志文件
  NameNode也记录着每个块所在的datanode的信息，但它并不保存块的位置信息，因为这些信息会在系统启动时，根据DataNode提供的信息来重建．
  NameNode容错：
    1.备份Snapshot和日志
    2.启动一个专门用于合并Snapshot和日志的被称作辅助NameNode的Server，并在NameNode发生故障时备用．
- Blocking Cache:
  By default, a block is cached in only one datanode's memory, although the number is configurable on a per-file basis.
  Users or applications can instruct the namenode which files to cache(and for how long) by adding a cache directive to a cache pool.
- HDFS Fedoration
  Split the namenode to different namenode which maintains part of file system namespace.

Hadoop has an abstract notion of filesystems, of which HDFS is just an implementation.

Hadoop Filesystems:
![](http://upload-images.jianshu.io/upload_images/4108852-cb37f41ec8174ceb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4108852-fd3083e18c3932a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Access HDFS by HTTP:

![](http://upload-images.jianshu.io/upload_images/4108852-f9cee889ba3ebd5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Read data from HDFS:
- Use Java's 'URL.openStream()' and HDFS's 'FsUrlStreamHandlerFactory'.
Disadvantage:
URL's setURLStreamHandlerFactory() method can be called only once in same JVM, so, if other libraries has called their method, here we will failed.
- Use Hadoop's FileSystem
- Random Access
  1. InputStream.seek()
  2. PositionedReadable.read(long position, byte[] buffer, int offset, int length)
  3. PositionedReadable.readFully(long position, byte[] buffer, int offset, int length)
  4. PositionedReadable.readFully(long position, byte[] buffer)
  Method 2, 3, 4 is thread-safe.
  Calling seek() is a relatively expensive operation and should be done sparingly. You should structure your application access pattern to rely on streaming data rather than performing a large number of seeks.

Writing Data:
- FileSystem.create(Path f)
  Other overloaded versions:
  1.Specify whether to forcibly over write existing file
  2.The replication factor of the file
  3.The buffer size to use when writing the file
  4.The block size for the file
  5.File permissions
  6.Accept a callable 'Progressable' so your application can be notified of the progress of the data being written to the data nodes.
- FileSystem.append(Path f)
  The append() operation is optional and not implemented by all Hadoop filesystems. For example, HDFS supports append, but S3 filesystems don't.
  Currently, none of the other Hadoop filesystems expect HDFS call progress() during writes.
- FSDataOutputStream
  'public long getPos()' method can querying the current position in the file.
  Howerver, unlike 'FSDataInputStream', 'FSDataOutputStream' does not permit seeking.
- FileSystem.mkdirs(Path f)
  Create a directory, and can create parent factory if necessary. Return true if all directory has been created.
- Listing files
  public FileStatus[] listStatus(Path f) throws IOException
  public FileStatus[] listStatus(Path f, PathFilter filter) throws IOException
  public FileStatus[] listStatus(Path[] files) throws IOException
  public FileStatus[] listStatus(Path[] files, PathFilter filter) throws IOException
- File pattern
  public FileStatus[] globStatus(Path pathPattern) throws IOException
  public FileStatus[] globStatus(Path pathPattern, PathFilter filter) throws IOException
- Deleting Data
  public boolean delete(Path f, boolean recursive)

Data flow of reading operation:
![](http://upload-images.jianshu.io/upload_images/4108852-6d76d8ccd4a81181.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Data flow of writing operation:
![](http://upload-images.jianshu.io/upload_images/4108852-304dd46fd98892e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Replica Place Strategy:
  A tradeoff between read-bandwidth, write-bandwidth and fault tolerance.
  Hadoop's fault strategy:
    - Place the first replica on the same node as the client(for clients running outside the cluster, a node is chosen at random, although the system tries not to pick nodes that are too full or too busy)
    - Place the second replica on a different rack as the first
    - Place the third replica on the same rack as the second, but on a different node chosen at random
    - Further replicas are placed on random nodes in the cluster, although the system tries to avoid placing too many replicas on the same rack.

Network distance in Hadoop:
For example, image a node **n1** on  rack **r1** in data center **d1**. This can be represented as **/d1/r1/n1**. Using this notation, here are the distances for the four scenarios:
  -distance(/d1/r1/n1, /d1/r1/n1)=0(processes on the same node)
  -distance(/d1/r1/n1, /d1/r1/n2)=2(different nodes on the same rack)
  -distance(/d1/r1/n1, /d1/r2/r3)=4(nodes on different racks in the same data center)
  -distance(/d1/r1/n1, /d2/r3/d4)=6(nodes in different data center)

HDFS Coherency Model:
HDFS supports final consistency by default.
When you call flush(), it returns after the first replicas has been flushed.
If we wanna get strong consistency, we can call hflush(), this method will return after flushing all replicas. But it doesn't guarantee that the datanodes have written the data to disk, only that it's in the datanode's memory.
For this stronger guaranee, use hsync() instead.

YARN:
- ResourceManager: Manage the use of resources across the cluster
- NodeManager: Launch and monitor containers. Dpending on how YARN is configured, a container may be a Unix process or a Linux group.

How YARN runs an application:
![](https://upload-images.jianshu.io/upload_images/4108852-277b85ccb377663f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

YARN itself does not provide any way for the parts of the application to communicate with one another.

YARN is just a resource manager which is responsible to allocate resource, lease resource and monitor resource. In MapReduce, it doesn't make sure how many mapper and reducer should be created, and how these components communicate with each other. It just allocate and release resource.

Application Lifespan:
- One application per user job, which is the approach that MapReduce takes
- One application per workflow or user session of jobs. Spark is an example that uses this model.
- Ａ long-running application that is shared by different users. Such an application often acts in some kinds of coordination role.

A comparison of MapReduce 1 and YARN components:

MapReduce 1 | YARN
--- | ---
JobTracker | ResourceManager, ApplicationMaster, TimelineServer
TaskTracker | NodeManager
slot | Container

YARN Schedule Policy:
- FIFO
  Disadvantage: Large applications will use all the resources in a cluster, so each application has to wait its turn.
- Capacity:
  Leave some space in advance.
  Advantage: Little job can be ran although big job is running
  Disadvantage: Waste resource
- Fair:
  Adjust resource dynamatically.
  Advantage: Use resource fully.
  Disadvantage: How to implement it?

CapacityScheduler:
  Each organization is allocated a certain capacity of the overall cluster.
  Each organization is set up with a dedicated queue that is configured to use a given fraction of the cluster capacity.
  Queue may be further divided in hierarchical fashion.
  In normal operation, the CapacityScheduler does not preempt containers by forcibly killing them.
  The job in each queue is scheduled by FIFO.

Hadoop Questions:
1. Does checksum will be calculated before data reaches datanode for storage?
  Yes, an end-to-end checksum calculation is performed as part of the HDFS write pipeline while the block is being written to DataNodes.
2. Why only last node should verify the checksum, Bit rot error can happen even in the initial data nodes as well while only last node has to verify it?
  The intent of the checksum calculation in the write pipeline is to verify the data in transit over the network, not check bit rot on disk. Therefore, verification at the final node in the write pipeline is sufficient. Checking for bit rot in existing replicas on disk is performed separately at each DataNode by a background thread.
3. Will checksum of the data is stored at datanode along with the checksum during WRITE process?
  Yes, the checksum is persisted at the DataNode. For each block replica hosts by a DataNode, there is a corresponding metadata file that can contains metadata about the replica, including its checksum information. The metadata file will have the same base name as the block file, and it will have an extension of ".meta"
4. Suppose i have a file of size 10MB, as per above statement there will be 20 checksums which will get created, if suppose block size is 1MB then as per i understood checksum has to be stored along with the block. So in this case each block will store 2 checksums with it?
  The DataNode stores a single ".meta" file corresponding to each block replica. Within that metadata file, there is an internal data format for storage of multiple checksums of different byte range within that block replica. All checksums for all byte ranges must be valid in order for HDFS to consider the replica to be valid.
5. For the above log file in datanode, will writes happen only when client sends successful msg. What if client ovserve failures in checksum calculation？
  This only logs checksum verification failures that are detected in the background by the DataNode. If a client detects a checksum failure at read time, then the client reports the failure to the NameNode, which then recovers by invalidating the corrupt replica and scheduling re-replication from another known good replica. There would be some logging in the NameNode log related to this activity.

Hadoop FileSystem:
- Local FileSystem
- RawLocalFileSystem: Does not execute file checksum
- ChecksumFileSystem

Compression:

Compression format | Tool | Algorithm | Filename extension | Splittable
--- | --- | --- | --- | ---
DEFLATE | N/A | DEFLATE | .deflate | No
gzip | gzip | DEFLATE | .gz | No
bzip2 | bzip2 | bzip2 | .bz2 | Yes
LZO | LZOP | LZO | .lzo | No
LZ4 | N/A | LZ4 | .lz4 | No
Snappy | N/A | Snappy | .snappy | No

gzip is a general compressor and sits in the middle of the space/time trade-off.

Compression format | Hadoop CompressionCodec
--- | ---
DEFLATE | org.apache.hadoop.io.compress.DefaultCodec
gzip | org.apache.hadoop.io.compress.GzipCodec
bzip2 | org.apache.hadoop.io.compress.BZip2Codec
LZO | com.hadoop.compression.lzo.LzopCodec
LZ4 | org.apache.hadoop.io.compress.Lz4Codec
Snappy | org.apache.hadoop.io.compress.SnappyCodec

MapReduce compression properties:

Property name | Type | Default value | Description
--- | --- | --- | ---
mapreduce.output.fileoutputformat.compress | boolean | false | Whether to compress outputs
mapreduce.output.fileoutputformat.compress.codec | class name | org.apache.hadoop.io.compress.DefaultCodec | The compression codec to use for outputs
mapreduce.output.fileoutputformat.compress.type | String | RECORD | The type of compression to use for sequence file outputs: NONE, RECORD, or BLOCK

Map output compression properties:

Property name | Type | Default Value | Description
--- | --- | --- | ---
mapreduce.map.output.compress | boolean | false | Whether to compress map outputs
mapreduce.map.output.compress.codec | Class | org.apache.hadoop.io.compress.DefaultCodec | The compression codec to use for map outputs

The internal structure of a sequence file with no compression and with record compression:

![](https://upload-images.jianshu.io/upload_images/4108852-a0bb1ff4b6b18122.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The internal structure of a sequence file with block compression:
![](https://upload-images.jianshu.io/upload_images/4108852-96ae8775552cf333.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Hadoop MapReduce accesses local files:
- By LocalFileSystem
- By ToolRunner and "-fs file:/// -jt local" option

Test MapReduce:
- Mapper: By **MapDriver** in Unit Test
- Reducer: By **ReduceDriver** in Unit Test
- Job Runner:
  - Uses the local filesystem and the local job runner, by setting 'fs.defaultFS' and 'mapreduce.framework.name'
  - Testing the driver by using "mini-" cluster

The client classpath:
- The job JAR file
- Any JAR files in the lib directory of the job JAR file, and the classes directory
- The classpath defined by HADOOP_CLASSPATH, if set

The task classpath:
- The job JAR file
- Any JAR files contained in the 'lib' directory of the job JAR file, and the classes directory
- Any files added to the distributed cache using the -libjars option, or the addFileToClassPath() method on DistributedCache, or Job

Task classpath precedences:
On the client side, you can force Hadoop to put the user classpath first in the search order by setting the HADOOP_USER_CLASSPATH_FIRST environment variable to true. For the task classpath, you can set "mapreduce.job.user.classpath.first" to true.

Job, Task and TaskAttempt IDs:
- Application: application_${timestamp_the_application}_${ApplicationID}
- Job: job_${timestamp_the_job}_${ApplicationID}
- Task: task_${timestamp_the_task}_${ApplicationID}_${job_type}_${TaskID}
- TaskAttemptID: attempt_${timestamp}_${ApplicationID}_${job_type}_${TaskID}_${Attempt_Time}

ResourceManager web ui:
http://resource-manager-host:8088/

Stastics the malformed record:
- Write an enum firstly:
  ~~~
    enum Temperature {
      MALFORMED
    }
  ~~~
- Increment it by context's counter:
  ~~~
    context.getCounter(Temperature.MALFORMED).increment()
  ~~~

Types of Hadoop logs:

Logs | Primary Audience | Description
--- | --- | ---
System daemon logs | Administrators | Each Hadoop daemon produces a logfile(using log4j) and another file that combines standard out and error. Written in the directory defined by HADOOP_LOG_DIR environment variable.
HDFS audit logs | Administrators | A log of all HDFS requests turned off by default. Written to the namenode's log, although this is configurable.
MapReduce job history logs | Users | A log of the events(such as task completion) that occur in the course of running a job. Saved centrally in HDFS.
MapReduce task logs | Users | Each task child process produces a logfile using log4j(called syslog), a file for data sent to standard out(stdout), and a file for standard error(stderr). Written in the userlogs subdirectory of the directory defined by the YARN_LOG_DIR environment variable.

Tuning checklist:

Area | Best practice
--- | ---
Number of mappers | How long are your mappers running for? If they are only running for a few seconds on average, you should see whether there's a way to have fewer mappers and make them all run longer- a minute or so, as a role of thumb. The extent to which this is possible depends on the input format you are using.
Number of reducers | Check that you are using more than a single reducer. Reduce tasks should run for five minutes or so and produce at least a block's worth of data, as a rule of thumb.
Combines |  Check whether your job can take advantage of a combiner to reduce the amount of data passing through the shuffle.
Intermediate compression | Job execution time can always benefit from enabling map output compression
Custom Serialization | If you are using your own custom writable objects or custom comparators, make sure you have implemented RawComparator
Shuffle tweaks | The MapReduce Shuffle exposes around a dozen tuning parameters for memory management, which may help you wring out the last bit of performance.

Job Control:
- Linear chain of Jobs
  ~~~
  JobClient.runJob(conf1)
  JobClient.runJob(conf2)
  ~~~
  - waitForCompletion()
- Directed acyclic graph of Jobs
  org.apache.hadoop.mapreduce.jobcontrol.JobControl

How Hadoop runs a MapReduce job:
![](https://upload-images.jianshu.io/upload_images/4108852-0d111aaa0542b098.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  The MapReduce application master, coordinates the tasks running the MapReduce job. The ApplicationMaster and the MapReduce tasks run in containers that are scheduled by the resource manager and managed by the node managers.

Job submission:
The job submission process implemented by JobSubmitter does the following:
  - Asks the resource manager for a new application ID, used for the MapReduce job ID
  - Checks the output specification of the job. For example, if the output directory has not been specified or it already exists, the job is not submitted and an error is thrown to the MapReduce program.
  - Computes the input splits for the Job. If the splits cannot be computed, the job is not submitted and an error is thrown to the MapReduce program.
  - Copies the resources needed to run the job, including the job JAR file, the configuration file, and the computed input splits, to the shared filesystem in a directory named after the job ID. The job JAR is copies with a high replication factor so that there are lots of copies across the cluster for the node managers to access when they run tasks for the job.
  - Submits the job by calling submitApplication() on the resource manager.

Job Initialization:
When the resource manager receives a call to its submitApplication() method, it hands off the request to the YARNScheduler. The scheduler allocates a container, and the resource manager then launches the application master's process there, under the node manager's management.
Next, it retrives the input splits computed in the client from the shared file system. It then creates a map task object for each split, as well as a number of reduce task objects determined by the `mapreduce.job.reduces` property. Tasks are given IDs at this point.
If the job is small, the application master may choose to run the tasks in the same JVM as itself. This happens when it judges that the overhead of allocating and running tasks in new containers outweighs the gain to be had in running them in parallel, compared to running them sequentially on one node.
By default, a small job is one that has less than 10 mappers, only one reducer, and an input size that is less than the size of one HDFS block.
Finally, before any tasks can be run, the ApplicationMaster calls the `setupJob()` method on the `OutputCommitter`. For FileOutputCommitter, which is the default, it will create the final output directory for the job and the temporary working space for the task output.

Task Assignment:
If the job does not qualify for  running as an uber task, then the application master requets containers for all the map and reduce tasks in the job from the ResourceManager.
Requests also specify memory requirements and CPUs for tasks. By default, each map and reduce task is allocated 1024MB of memory and one virtual core.

Task Execution:
Once a task has been assigned resources for a container on a particular mode by the ResourceManager's scheduler, the ApplicationMaster starts the container by contacting the NodeManager. The task is executed by a Java application whose main class is `YarnChild`. Before it can run the task, it localizes the resources that the task needs, including the job configuration and JAR file, and any files from the distribtued cache.

The relationship of the streaming executable to the NodeManager and the task container:
![](https://upload-images.jianshu.io/upload_images/4108852-a94d652dc32c9c1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Map-side tuning properties:
priority name | type | default vlaue | description
--- | ---  | --- | ---
mapreduce.task.io.sort.mb | int | 100 | The size, in MB, of the memory buffer to use while sorting map output
mapreduce.map.sort.spill.percent | float | 0.80 | The threshold usage proportion for both the map output memory buffer and the record boundaries index to start the process of spilling to disk
mapreduce.task.io.sort.factor | int | 10 | The maximum number of streams to merge at once when sorting files. This property is also used in the reduce. It's fairly common to increase this to 100.
mapreduce.map.combine.minspills | int | 3 | The minimum number of spill files needed for the combiner to run(If a combiner is specified)
mapreduce.map.output.compress | boolean | false | Whether to compress map outputs
mapreduce.map.output.compress.codec | Class name | org.apache.hadoop.io.compress.DefaultCodec | The compression codec to use for map outputs
mapreduce.shuffle.max.threads | int | 0 | The number of worker threads per node manager for serving the map outputs to reducers. This is a cluster wide setting and cann't be set by individual jobs. 0 means use the twice number of available processors.

Reduce-side tuning properties:

Property name | Type | Default value | Description
--- | --- | --- | ---
mapreduce.reduce.shuffle.parallelcopies | int | 5 | The number of threads used to copy map outputs to the reducer
mapreduce.reduce.shuffle.maxfetchfailures | int | 10 | The number of times a reducer tries to fetch a map output before reporting the error.
mapreduce.task.io.sort.factor | int |10 | The maximum number of streams to merge at once when sorting files. This property is also used in the map.
mapreduce.reduce.shuffle.input.buffer.percent | float | 0.70 | The proportion of total heap size to be allocated to the map outputs buffer during the copy phase of the shuffle.
mapreduce.reduce.shuffle.merge.percent | float | 0.66 | The threshold usage proportion for the map outputs buffer(defined by mapred.job.shuffle.input.buffer.percent) for starting the process of merging the outputs and spilling to disk
mapreduce.reduce.merge.inmem.threshold | int | 1000 | The threshold number of map outputs for starting the process of merging the output and spiling to disk. A value of 0 or less means there is no threshold, and the spill behaviour is governed solely by `mapreduce.reduce.shuffle.merge.percent`
mapreduce.reduce.input.buffer.percent | float | 0.0 | The proportion of total heap size to be used for retaining map outputs in memory during the reduce.

Task environment properties:

Property name | Type | Description | Example
--- | --- | --- | ---
mapreduce.job.id | String | The job ID | job_20081201130_0004
mapreduce.task.id | String | The task ID | task_200811201130_0004_m_000003
mapreduce.task.attempt.id | String | The task attempt ID | attempt_200811201130_0004_m_000003_0
mapreduce.task.partition | int | The index of the task within the job | 3
mapreduce.task.ismap | boolean | Whether this task is a map task | true

Speculative execution properties:

Property name | Type | Default value | Description
--- | --- | --- | ---
mapreduce.map.speculative | boolean | true | Whether extra instances of map tasks may be launched if a task is making slow progress
mapreduce.reduce.speculative | boolean | true | Whether extra instances of reduce tasks may be launched if a task is making slow progress
yarn.app.mapreduce.am.job.speculator.class | Class | org.apache.hadoop.mapreduce.v2.app.speculate.DefaultSpeculator | The Speculator class implementing the speculative execution policy.
yarn.app.mapreduce.am.job.task.estimator.class | Class | org.apache.hadoop.mapreduce.v2.app.speculate.LegacyTaskRuntimeEstimator | An implementation of TaskRuntimeEstimator used by Speculator instance that provides estimates for task runtimes.

Small files and CombineFileInputFormat:
Hadoop works better with  small number of large files than a large number of small files. One reason for this is that `FileInputFormat` generates splits in such a way that each split is all or part of a single file.
Where `FileInputFormat` creates a split per file, `CombineFileInputFormat` packs many files into each split so that each mapper has more to process.
The split size is calculated by the following formula:
**max(minimumSize, min(maximumSize, blockSize))**
and by default:
**minimumSize < blockSize < maximumSize**
One technique for avoiding the many small files case is to merge small files into larger files by using a sequence file.

Preventing splitting:
- Increase the minimum split size to be larger than the largest file in your system.
- Subclass the concrete subclass of `FileInputFormat` that you want to use, to override the `isSplitable()` method to return false.

File information in the mapper:
By calling the `getInputSplit()` method on the Mapper's context object.

InputFormat:
- Text input:
  - TextInputFormat
  - CombineFileInputFormat
  - KeyValueTextInputFormat
  - NLineInputFormat
- Binary input:
  - SequenceFileInputFormat
  - SequenceFileAsTextInputFormat
  - SequenceFileAsBinaryInputFormat
  - FixedLengthInputFormat

Multiple Inputs:
~~~
MultipleInputs.addInputPath(job, inputpath, TextInputFormat.class, Mapper.class);
~~~
The map outputs have the same types, since the reducers see the aggregated map outputs and are not aware of the different mappers used to produce them.

Database Input:
DBInputFormat: Reading data from a relational database, using JDBC. But it hasn't any sharding capabilities.

![](https://upload-images.jianshu.io/upload_images/4108852-621fc57747024d4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-1a393756d87c0f0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Output:
- TextOutput
  - TextOutputFormat
  - NullOutputFormat
- BinaryOutput
  - SequenceFileOutputFormat
  - SequenceFileAsBinaryOutputFormat
  - MapFileOutputFormat

Built-in counters:

Group | Name/Enum
--- | ---
MapReduce task counters | org.apache.hadoop.mapreduce.TaskCounter
FileSystem counters | org.apache.hadoop.mapreduce.FileSystemCounter
FileInputFormat counters | org.apache.hadoop.mapreduce.lib.input.FileInputFormatCounter
FileOutputFormat counters | org.apache.hadoop.mapreduce.lib.output.FileOutputFormatCounter
Job counters | org.apache.hadoop.mapreduce.JobCounter

Built-in MapReduce task counters:
  - Map input records
  - Split row bytes: The number of bytes of input-split objects read by maps. These objects represent the split metadata rather than the split data itself, so the total size should be small.
  - Map output records
  - Map output bytes
  - Map output materialized bytes: The number of bytes of map output actually written to disk. If map output compression is enabled, this is reflected in the counter value.
  - Combine input records
  - Combine output records
  - Reduce input groups: The number of distinct key groups consumed by all the reducers in the job. Incremented every time the collect() method is called on a combiner's OutputCollector
  - Reduce input records
  - Reduce output records
  - Reduce shuffle bytes
  - Spilled records: The number of records spilled to disk in all map and reduce tasks in the job
  - CPU milliseconds
  - Physical memory bytes
  - Virtual memory bytes
  - Committed heap bytes
  - GC time milliseconds
  - Shuffled maps
  - Failed shuffle
  - Merged map outputs

Built-in filesystem task counters:
- FileSystem bytes read
- FileSystem bytes written
- FileSystem read ops
- FileSystem large read ops
- FileSystem write ops

Built-in FileInputFormat task counters
- Bytes read: The number of bytes read by map tasks via the FileInputFormat

Built-in FileOutputFormat task counters
- Bytes written: The number of bytes written by map tasks(for map-only jobs) or reduce tasks via the FileOutputFormat.

Built-in job counters
- Launched map tasks
- Launched reduce tasks
- Launched uber tasks
- Maps in uber tasks
- Reduces in uber tasks
- Failed map tasks
- Failed reduce tasks
- Failed uber tasks
- Killed map tasks
- Killed reduce tasks
- Data-local map tasks
- Rack-local map tasks
- Other local map tasks
- Total time in map tasks
- Total time in reduce tasks

Sort by value
- Make the key a composite of the natural key and natural value
- The sort comparator should order by the composite key
- The partitioner and grouping comparator for the composite key should consider only the natural key for partitioning and group

Uber模式
以Uber模式运行MR作业，所有的MapTasks和ReduceTasks将会在ApplicationMaster所在的容器中运行，也就是说，整个MR作业运行的过程只会启动AM Container,因为不需要启动mapper和reducer Container.

实现排序：
1. 由于Mapper会对同一partition内的数据进行排序，所以只需要一个partition，一个reducer就能实现全局有序。但这并不能充分利用集群资源。
2. Sharding的做法，将小于k1的数据发送到一个partition，大于k1并小于k2的数据发送到另一个partition，大于k2的发送到最后一个partition.由于各paritition的数据已经排好序，所以将这几个partition按从小到大的规则进行排序即可实现全局有序。缺点是负载不一定均衡。
3. 类似第二种，配合InputSampler和TotalOrderPartitioner来实现

Join的实现：
1. 在Reduce端进行Join
  Map端的主要工作：为来自不同文件的key/value对打标签以区别不同来源的数据。然后用连接字段作为key，其余部分和新加的标志作为value,最后进行输出
  Reduce端的主要工作：将来源于不同文件的数据分开，最后进行笛卡尔积
2. 在Map端进行连接
  使用场景：一张表十分小，一张表十分大
  用法：在提交作业的时候，先将小表文件放到该作业的DistributedCache中，然后从DistributedCache中取出该小表进行Join。Key/Value分割放在内存中，然后扫描大表，看大表中每条记录的join key/value值是否能够在内存中找到相同join key的记录，如果有则直接输出结果。
3. 半连接
  类似第一种方法，但是在map阶段扫描连接表，将join key不在内存HashSet中的记录过滤掉，让那些参与join的记录通过Shuffle传输到Reducer端进行join操作，其他的和Reduce join都是一样的。

![](https://upload-images.jianshu.io/upload_images/4108852-4ab8aa395ef9dae6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-7be527597dc46f24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Important YARN daemon properties:

Property name | Type | Default value | Description
--- | --- | --- | ---
yarn.resourcemanager.hostname | Hostname | 0.0.0.0 | The hostname of the machine the resource manager runs on. Abbreviated  ${y.rm.hostname} below.
yarn.resourcemanager.address | Host name and port | ${y.rm.hostname}:8032 | The hostname and port that the resource manager's RPC server runs on
yarn.nodemanager.localdirs | Comma-separated directory names | ${hadoop.tmp.dir}/nm-local-dir | A list of directories where node managers allow container to store intermediate data. The data is cleared out when the application ends.
yarn.nodemanager.auxservices | Comma-separated service name | | A list of auxiliary services run by node manager. A service is implemented by the class defined by the property 'yarn.nodemanager.auxservices.servicename.class'. By default, no auxiliary services are specified.
yarn.nodemanager.resource.memorymb | int | 8192 | The amount of physical memory that may be allocated to containers being run by the node manager.
yarn.nodemanager.vmem-pmem-ratio | float | 2.1 | The ratio of virtual to physical memory for containers. Virtual memory usage may exceed the allocation by this amount.
yarn.nodemanager.resource.cpu-vcores | int | 3 | The number of CPU cores that may be allocated to containers being run by the nodemanager

![](https://upload-images.jianshu.io/upload_images/4108852-5d7f9e41f59e8878.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The checkpointing process:
1. The secondary asks the primary to roll its in-progress 'edits' file, so new edits go to a new file. The primary also updates the 'seen_txid' file in all its storage directories.
2. The secondary retries the latest 'fsimage' and 'edits' files from the primary
3. The secondary loads 'fsimage' into memory, applies each transaction from 'edits', then creates a new merged 'fsimage' file
4. The secondary sends the new 'fsimage' back to the primary, and the primary saves it as a temporary '.ckpt' file
5. The primary renames the temporary 'fsimage' file to make it available
