---
layout: post
title: 《Kafka--The-Definitive-Guide》读书笔记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
A topic typically has multiple partitions, there is no guarantee of message time - ordering across the entire topic, just within a single partition.

## Broker

A single Kafka server is called a broker. The broker receives messages from producers assign offsets to them, and commits the messages to storage on disk. It also services consumers, responding to fetch requests for partitions and responding with the messages that have been committed to disk.

A partition is owned by a single broker in the cluster, and that broker is called the leader of the partition. A partition may be assigned to multiple brokers, which will result in the partition being replicated.

![](https://upload-images.jianshu.io/upload_images/4108852-efb2a929099602b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The message will be placed in a buffer and will be sent to the broker in a separate thread.

We start producing messages to Kafka by creating a `ProducerRecord`, which must in dude the topic we want to send the record to and a value. Optionally, we can also specify a key and/or a partition. Once we send the `ProducerRecord`, the first thing the producer will do is serialize the key and value objects to ByteArrays so they can be sent over the network.

Next, the data is sent to a partitioner. If we specified a partition in the `ProducerRecord`, the partitioner doesn't do anything and simply returns the partition we specified. If we didn't, the partitioner will choose a partition for us, usually based on the `ProducerRecord` key. Once a partition is selected, the producer knows which topic and partition and record will go to. It then adds the record to a batch of records that will also be sent to the same topic and partition. A separate thread is responsible for sendidng those batches of records to the appropriate Kafka brokers.

When the broker receives the messages, it sends back a response. If the messages were successfully written to Kafka, it will return a `ReocrdMetadata` object with the topic, partition, and the offset of the record within the partition. If the broker failed to write the messages, it will return an error. When the producer receives an error, it may retry sending the message a few more times before giving up and returning an error.

## Constructing a Kafka Producer

- bootstrap.servers: List of host:port pair of brokers that the producer will use to establish initial connection to the Kafka cluster
- key.serializer: Name of a class that will be used to serialize the keys of the records we will produce to Kafka.
- value.serializer: Name of a class that will be used to serialize the values of the records we will produce to Kafka.

There are tree primary methods of sending messages:
- Fire-and-forget:
  We send a message to the server and don't really care if it arrives successfully or not. Most of the time, it will arrive successfully, since Kafka is highly available and the producer will retry sending messages automatically. However, some messages will get lost using this method.
- Synchronous send:
  We send a message, the send() method returns a `Future` object, and we use `get()` to wait on the future and see if the send() was successful or not.
- Asynchronous sned:
  We call the `send()` method with a callback funciton, which gets triggered when it receives a response from the Kafka broker.

Keys server two goals: They are additional information that gets stored with the message, and they are also used to decide which one of the topic partitions the message will be written to.

When the key is null and the default partitioner is used, the record will be send to one of the available partitions of the topic at random.

There is no point in adding more consumers than you have partitions in a topic-some of the consumers will just be idle.

You create a new consumer group for each application that needs all the messages from one or more topics. You add consumers to an existing consumer group to scale the reading and processing of messages from the topics, so each additional consumer in a group will only get a subset of the messages.

Moving partition ownership from one consumer to another is called a `rebalance`. During a rebalance, consumers can't consume messages, so a rebalance is basically a short window of unavailability of the entire consumer group. In addition, when partitions are moved from one consumer to another, the cunsumer loses its current state; if it was caching any data, it will need to refresh its caches - slowing down the application until the consumer sets up its state again.

## How does the process of assigning partitions to brokers work?

When a consumer wants to join a group, it sends a `JoinGroup` request to the group coordinator. The first consumer to join the group becomes the group leader. The leader receives a list of all consumers in the group from the group coordinator(this will include all consumers that sent a heartbeat recently and which are therefore considered alive) and is responsible for assigning a subset of partitions to each consumer. It uses an implementation of `PartitionAssigner` to decide which partitions should be handled by which consumer.

Kafka has two built-in partition assignment policies. After deciding on the partition assignment, the consumer leader sends the list of assignments to the `GroupCoordinator`, which sends this information to all consumers. Each consumer only sees his own assignment - the leader is the only client process that has the full list of consumers in the group and their assignments. This process repeats every time a rebalance happens.

## Commits and Offsets

How does a consumer commit an offset? It produces a message to Kafka, to a special `--consumer-offsets` topic, with the committed offset for each partition. As long as all your consumers are up, running, and churning away, this will have no impact. However, if a consumer crashes or a new consumer joins the consumer group, this will trigger a rebalance. After a rebalance, each consumer may be assigned a new set of partitions than the one it processed before. In order to know where to pick up the work, the consumer will read the latest committed offset of each partition and continue from there.

If the committed offset is smaller than the offset of the last message the client processed, the messages between the last processed offset and the committed offset will beprocessed twice.

If the committted offset is larger than the offset of the last message the client actually processed, all messages between the last processed offset and the committed offset will be missed by the consumer group.

## Automatic Commit

If you configure `enable.auto.commit=true`, then every five seconds the consumer will commit the largest offset your client received from `poll()`. The five-second interval is the default and is controlled by setting `auto.commit.interval.ms`.

With autocommit enabled, a call to poll will always commit the last offset returned by the previous poll. It doesn't know which events were actually processed, so it is critical to always process all the events returned by `poll()` before calling `poll()` again.

`commitSync()` will commit the latest offset returned by `poll()` and return once the offset is committed, throwing an exception if commit fails for some reason.

`commitSync()` will commit the latest offset returned by poll(), some make sure you call commitSync() after you are done processing all the records in the collection.

`commitSync()` will retry the commit until it either succeed or encounters a nonretriable failure. `commitAysnc()` will not retry.

A common pattern is to combine `commitAsync()` with `commitSync()` just before shutdown. 

## Commit specific offset

Call `commitSync()` and `commitAsync()` with parameter whose type is `Map<TopicPartition, OffsetAndMetadata>`.

## Rebalance Listeners

The consumer API allows you to run your own code when partitions are added or removed from the consumer. You do this by passing a `ConsumerRebalanceListener` when calling the `subscribe()` method we discussed previously. `ConsumerRebalanceListener` has two methods you can implement:
- onPartitionsRevoked(Collection<TopicPartition> partitions)
Called before the rebalancing starts and after the consumer stopped consuming messages. This is where you want to commit offsets, so whoever gets this partition next will know where to start.
- onPartitionsAssigned(Collection<TopicPartition> partitions)
Called after partitions have been reassigned to the broker, but before the consumer starts consuming messages.

When you decide to exit the poll loop, you will need another thread to call `consumer.wakeup()`. If you are running the consumer loop in the main thread, this can be done from shutdown Hook. Note that consumer.wakeup() is the only consumer method that is safe to call from different thread. Calling makeup will cause `poll()` to exit with `WakeupException`, or if `consumer.wakeup()` was called while the thread aas not waiting on poll, the exception will be thrown on the next iteration when poll() is called. The `WakeupException` doesn't need to be handled, but before exiting the thread, you must call `consumer.close()`.

## Cluster Membership

Kafka uses Apache Zookeeper to maintain the list of brokers that are currently numbers of a cluster. Every broker has a unique idetifier that is either set in the broker configuration file or automatically generated. Everytime a broker process starts, it registers itself with its ID in ZooKeeper by creating an epheneral node. Different Kafka components subscribe to the `/brokers/ids` path in ZooKeeper where brokers are registered so they get notified when brokers are added or removed.

## The Controller

Kafka uses ZooKeeper's ephemeral node feature to elect a controller and to notify the controller when nodes join and leave the cluster. The controller is responsible for electing leaders among the partitions and replicas whenever it notifies nodes join and leave the cluster. The controller uses the epoch number to prevent a `split brain` scenario where two nodes believe each is the current controller.

## Replication

Each topic is partitioned, and each partition can have multiple replicas. Those replicas are stored on brokers, and each broker typically stores hundreds or even thousands of replicas belonging to different topic and partitions.

There are two types of replicas:
- Leader replica: Each partition has a single replica designated as the leader. All produce and consume requests go through the leader, in order to guarantee consistency.
- Follower replica: All replicas for a partition that are not leaders are called followers. Followers don't serve client requests; their only job is to replicate messages from the leader and stay up-to-date with the most recent messages the leader has. In the event that a leader replica for a partition crashes, one of the follower replicas will be promoted to become the new leader for the partition.

## Request Processing

Most of what a Kafka broker does is process requests sent to the partition leaders from clients, partition replicas, and the controller. Kafka has a binary protocol that specifies the format of the requests and how brokers respond to them - both when the request is processed successfully or when the broker encounters errors while processing the request. All requests sent to the broker from a specific client will be processed in the order in which they were received - this guarantee is what allow Kafka to behave as a message queue and provide ordering guarantees on the messages it stores.

All requests have a standard header that includes:
- Request type
- Request version
- Correlation ID
- Client ID

For each port the broker listens on, the broker runs an `acceptor` thread that creates a connection and hands it over to a `processor` thread for handling. The number of processor threads is configurable. The network threads are responsible for takeing requests from client connections, placing them in a `request queue`, and picking up responses from a `response queue` and sending them back to clients.

Once requests are placed on the request queue, IO threads are responsible for picking them up and processing them. The most common types of requests are:
- Produce requests: Sent by producers and contain messages the clients write to Kafka brokers
- Fetch requests: Sent by consumers and follower replicas when they read messages from Kafka brokers.

How does the clients know where to send the requests? Kafka clients use another request type called a `metadata request`, which includes a list of topics the client is insterested in. The server response specifies which partitions exist in the topics, the replicas for each partition, and which replica is the leader. Metadata requests can be sent to any broker because all brokers have a metadata cache that contains this information.

## Produce Requests

When the broker that contains the lead replicas for a partition receives a produce request for this partition, it will start by running a few validations:
- Does the user sending the data have write privileges on the topic?
- Is the number of `ack` specified in the request valid?
- If acks is set to `all`, are there enough in-sync replicas for safely writing the message?

Then it will write the new messages to local disk. On Linux, the messages are written to the filesystem cache and there is no guarantee about when they will be written to disk. Kafka does not wait for the data to get  persisted to disk - it relies on replication for message durability.

Once the message is written to the leader of the partition, the broker exiamines the `ack` configuration - if acks is set to 0 or 1, the broker will respond immediately, if `acks` is set to `all`, the request will be stored in a buffer called `purgatory` until the leader observes that the follower replicas replicated the message, at which point a response is sent to the client.

## Partition Allocation

When doing the allocations, the goals are:
- To spread replicas evenly among brokers
- To make sure that for each partition, each replica is on a different broker
- If the brokers have rack information, then assign the replicas for each partition to different racks if possible.

## File Management

Partition is divided into `segments`. By default, each segment contains either 1GB of data or a week of data, whichever is smaller. As a Kafka broker is writing to a partition, if the segment limit is reached, we close the file and start a new one.

## File Format

Each segment is stored in a single data file. Inside the file, we store Kafka messages and their offsets. The format of the data on the disk is identical to the format of the messages that we send from the producer to the broker and later from the broker to the consumer. Using the same message format on disk and over the wire is what allows Kafka to use zero-copy optimization when sending messages to consumer and also avoid decompressing and recompressing messages that the producer already compressed.

Each message contains:
- key
- value
- offset
- message size
- checksum code
- magic byte
- compression codec
- timestamp

If you are using compression on the producer, sending larger batches means better compression both over the network and on the broker disks.

## Indexes

In order to help brokers quickly locate the message for a given offset, Kafka maintains an index for each partition. The index maps offsets to segment files and positions within the file.

Indexes are also broken into segments, so we can delete old index entries when the messages are purged. Kafka does not attempt to maintain checksums of the index. If the index becomes interrrupted, it will get regenereated from the matching log segment simply by rereading the messages and recording the offsets and locations.

## Compaction

Only stores the most recent value for each key in the topic.

If compaction is enabled when Kafka starts each broker will start a compaction manager thread and a number of compaction threads. These are responsible for performing the compaction tasks. Each of these threads chooses the partition with the highest ratio of dirty messages to total partition size and cleans this partition.

Compact the dirty log and merge with clena logs.

## Deleted Event

Compact it with `null` value, as a ttombstone flag.

## When are topics compacted?

Messages are eligble for compaction only on inactive segments.

In version 0.10.0 and older, Kafka will start compacting when 50% of the topic contains dirty records.

## What does Apache Kafka guarantee?

- Kafka providers order guarantee of messages in a partition
- Produced messages are considered `committed` when they were written to the partition on all its in-sync replicas.
- Messages that are committed will not be lost as long as at least one replica remains alive
- Consumers can only read messages that are committed.

## Reliable Producer

There are two important things that everyone who writes applications that produce to Kafka must pay attention to:
- Use correct `acks` configuration to match reliability requirements
- Handle errors correctly both in configuration and in code

The principles about tuning Kafka for cross-data center communication:
- No less than one cluster per datacenter
- Replicate each event exactly once(barring retires due to errors) between each pair of data centers
- When possible, consume from a remote data center rather than produce to a remote datacenter.

## Hub-and-Spokes Architecture

![](https://upload-images.jianshu.io/upload_images/4108852-297fda1269ae693e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

This architecture is used when data is produced in multiple datacenters and some consumers need access to entire data set. The architecture also allows for applications in each data center to only process data local to that specific datacenter. But it does not give access to the entire data set from each datacenter.

Use of this pattern is usually limited to only parts of the data set that can be completely separated between regional data centers.

## Active-Active Architecture

This architecture is used when two or more data centers share some or all of the data and each data center is able to both produce and consume events.

![](https://upload-images.jianshu.io/upload_images/4108852-8014123ed80818a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Benefit:
- Redundancy

Drawback:
- Avoid conflict
- data consistency

## Active-Standby Architecture

![](https://upload-images.jianshu.io/upload_images/4108852-112edd65abe28bbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The data in `fallover kafka cluster` is not always latest when failover.

Start offset for applications after failover:
- Auto Offset reset
- Replicate offsets topic 
- Time-based failover
- External offset mapping

## MirrorMaker

![](https://upload-images.jianshu.io/upload_images/4108852-fb5fbd3975bd0a17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Administering Kafka

- kafka-topics.sh: Provides easy access to most topic operations
- kafka-consumer-groups.sh: Manage consumer group
- kafka-config.sh: Manage configuration
- kafka-preferred-replica-election.sh: Start a preferred replica election for all topics in a cluster
- kafka-replica-verification.sh
