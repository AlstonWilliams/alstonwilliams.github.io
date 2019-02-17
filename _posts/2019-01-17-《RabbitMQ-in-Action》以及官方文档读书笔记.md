---
layout: post
title: 《RabbitMQ-in-Action》以及官方文档读书笔记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
## Official documentation
1. Round-robin dispatching:
  Every node will get the same number of messages.
  But the nodes are different actually, some are light, and some are heavy. How to assign more messages to light nodes, and less to heavy nodes?

2. Message acknowledgement
  There aren't any message timeouts; RabbitMQ will redeliver the message when the consumer dies. It's time even if processing a message a very very long time.
  Manual message acknowledgements are turn on by default. And auto message acknowledgements are turn off.

3. Message durability
  Two things are required to make sure that messages are not lost: Mark both the queue and messages are durable.
  The persistence guarantees are't strong because of disk cache. If you need a stronger guarantee then you can use "publisher confirm".

4. Fair dispatch
  Assume we have two workers, and all odd messages are heavy and even messages are light. Then one worker will be constantly busy and the other one will do hardly any work.
  In order to detect that we can use the `basicQos` method with the `prefetchCount=1` setting. This tells RabbitMQ server don't dispatch a new message to a worker until it has processed and acknowledged the previous one.

5. Exchange
  Determine how producer pushs messages to queue, fanout or other ways?

6. Temporary queue
  The queue will be delete after losing connection.

7. Binding
  Bind queue with exchange.

8. Routing
  Producer publishes messages by routing key, consumer binds queue to exchange by routing key.
  Routing key in some exchange types is not available, like "fanout". But "direct" exchange can work.

9. Exchange type:
  - fanout
  - direct
  - topic
    Like regular expression, consumer consumes message from queue by regular expression.
    `*` can substitute for exactly one word.
    `#` can substitute for zero or more words.
    The content between two dot is considered as a word.

## RabbitMQ in Action

#### Queue

1. Consumers receive messages from a particular queue in one of two ways:
  1.1 By subscribing to it via the `basic.consume` AMQP command. This will place the channel being used into a receive mode until unsubscribed from the queue. While subscribed, your consumer will automatically receive another message from the queue after consuming the last received message. You should use `basic.consume` if your consumer is processing many messages out of a queue and/or needs to automatically receive messages from a queue as soom as they arrive.
  1.2 Sometimes, you just want a single message from a queue and don't need to be persistently subscribed. Requesting a single message from the queue is done by using the `basic.get` AMQP command. This will cause the consumer to receive the next message in the queue and then not receive further messages until the next `basic.get`. You shouldn't use `basic.get` in a loop as an alternative to `basic.consume`, because it's much more intensive on Rabbit.

2. When a RabbitMQ queue has multiple consumers, messages received by the queue are served in a round-robin fashion to the consumers. Each message is sent to only one consumer subscribed to the queue.

3. Every message that's received by a consumer is required to be acknowledged. Either the consumer must explicitly send an acknowledgement to RabbitMQ using the `basic.ack` AMQP command, or it can set the `auto_ack` parameter to true when it subscribes to the queue.

4. Reject a message
  4.1 From all versions, you can disconnect the consumer. The disadvantage is the extra load put on the RabbitMQ server from the connecting and disconnecting of your consumer.
  4.2 If you're running RabbitMQ 2.0.0 or newer, use the `basic.reject` AMQP command.

5. Both consumers and producers can create queues by using the `queue.declare` AMQP command. But consumers can't declare a queue while subscribed to another one on the same channel. They must first unsubscribe in order to place the channel in a `transmit` mode.

6. Queue's properties
  6.1 exclusive: When set to true, your quue becomes private and can only be consumed by your app.
  6.2 auto-delete: The queue is automatically deleted when the last consumer unsubscribes.

7. Messages  that get published into an exchange but have no queue to be routed to are discarded by Rabbit. So if you can't afford for your messages to be black-holed, both your producers and your consumers should attempt to create the queues that will be needed.

#### Exchange and bindings

1. Exchange Type:
  - direct
  - fanout
  - topic
  - headers

`headers` exchange allows you to match against a header in the AMQP message instead of the routing key. Other than that, it operates identically to the direct exchange but with worse performance. As a result, it doesn't provide much real-world benefit and is almost never used.

   `direct` exchange is pretty simple: If the routing key matches, then the message is delivered to the corresponding queue.
  
  When a queue is declared, it will be automatically bound to that exchange using the queue name as routing key.

  `fanout` exchange is simple: When you send a message to a fanout exchange, it will be delivered to all the queues attached to this exchange.

#### Message durability

1. For a message that's in flight inside RabbitMQ to survive a crash, the message must:
  - Have its delivery mode option set to 2(persistent)
  - Be published into a durable exchange
  - Arrive in a durable queue.

2. The way that RabbitMQ ensures persistent messages survive a restart is by writing them to the disk inside of a persistency log file. When you publish a persistent message to a durable exchange, Rabbit won't send the response until the message is committed to the log file. Keep in mind, though, that if it gets routed to a nondurable queue after that, it's automatically removed from the persistency log and won't survive a restart. When you use persistent messages it's crucial that you make sure all three elements required for a message to persist are in place. Once you consume a persistent message from a durable queue, RabbitMQ flags it in the persistency log for garbage collection. If RabbitMQ restarts anytime before you consume a persistent message, it'll automatically re-create the exchanges and queues(and bindings) and replay any messages in the persistency log into the appropriate queues or exchanges.

3. The performance will be decreased if you persistent all your messages.

4. Though RabbitMQ clustering allows you to talk to any queue present in the cluster from any node, those queues are actually evenlly distributed among the nodes without redundancy. If the cluster node hosting your `seed_bin` queue crashes, the queue disappears from the cluster until the node is restored If the queue was durable.  More important, while the node is down its queues are't available and the durable ones can't be re-created. This can lead to black-holing of messages.

5. When should you use persistent/durable messaging? If you require high performance, for example, 100000 messages per second on a single server ,you should probably look at other ways to ensuring message delivery. For example, your producer could listen to a reply queue on a separate channel. Every time it publishes a message, it includes the name of the reply queue so that the consumer can send a reply back to confirm the receipt. If a message isn't replied to within a reasonable amount of time, the producer can republish the message. That said, the critical nature of messages requiring guaranteed delivery generally means they're slower in volumes than other types of messages.

6. We're just selective about what types of content use persistent messaging. For example, we run two types of RabbitMQ clusters: traditional RabbitMQ clustering for nonpersistent messaging, and pairs ofactive/hot-standby nonclustered RabbitMQ servers for persistent messaging.

7. Do keep in mind that while RabbitMQ can help ensure delivery, it can never absolutely guarantee it. Hard driver corruption, buggy behaviour by a consumer, or other extreme events can trash/black-hole persistent messages.

8. When you absolutely need to be sure the broker has the message in custody before you moving into another task, you need to wrap it in a transaction. Don't confuse the transtion in databases and in AMQP. In AMQP, after you place a channel into transaction mode, you send it the publsih you want to confirm, followed by zero or more other AMQP commands that should be executed or ignored depending on whether the initial publish succeed. Once you've sent all of the commands, you commit the transaction. If the transaction's initial publish succeds, then the channel will complete the other AMQP commands in the transaction. If the publish fails, none of the other AMQP commands will be executed.

9. Though transactions are part of the formal AMQP 0-9-1 specification, they have an Achilles heels in that they're huge drains on RabbitMQ performance. Not only can using transactions drop your message throught put by a factor of 2-10x, but they also make your producer app synchronous.

10. To ensure message delivery, the guys at RabbitMQ decides to come up a better concept: producer confirms. Same to transactions, you have to tell RabbitMQ to place the channel into confirm mode, and you can't turn it off without re-creating the channel. Once a channel is in confirm mode, every message published on the channel will be assigned a unique ID number. Once the message has been delivered to all queues that have bindings matching the message's routing key, the channel will issue a publisher confirm to the producer app. The major benefit of publisher confirms is that they're asynchronous. Once a message has been published, the producer app can go on to the next message while waiting for the confirm. When the confirm for that message is finally received, a callback function in the producer app will be fired so it can make up and handle the confirmation. If an internal error occurs inside Rabbit taht causes a message to be lost, RabbitMQ will send a message "nack" that's like a publisher confirm but indicates the message was lost. Also, since there's no concept of message rollback, publisher confrms are much ligher weight and have an almost negligible performance hit on the RabbitMQ broker.

#### Running and administering Rabbit

1. The metadata for every queue, exchange, and binding in RabbitMQ(but not message content) is written to Mnesia, which is a non-SQL database built light into Erlang.

2. Mnesia configuration options:
- dump_log_write_threshold: How offen to flush/dump the contents of the append-only log into the actual database files. It's specified in terms of hwo many entries must be in the log before a dump operation occurs. Setting this to a higher number can reduce I/O load and increase performance for persistent messages.

3. Rabbit configuration options
- tcp_listeners: Defines which IP addresses and ports RabbitMQ should listen on for non-ssl encrypted traffic.
- ssl_listeners: Defines which IP addresses and ports RabbitMQ should listen on for ssl-encrypted traffic.
- ssl_options: Specifies SSL-related options. Valid options are "cacertfile"(CA certificate file), "certfile"(server certificate file), "keyfile"(server key file) and "fail_if_no_peer_cert"(require client to have a valid certificate: True/ False)
- vm_memory_high_watermark: Controls how much memory RAM RabbitMQ is allowed to consume
- msg_store_file_size_limit: The maximum size of the message store DB before RabbitMQ garbage collects the contents of the store.
- queue_index_max_journal_entries: The maximum number of entries in the message store journal before they're flushed into the message store DB and committed.

4. User-related operation:
- rabbitmqctl add_user
- rabbitmqctl delete_user
- rabbitmqctl list_users
- rabbitmqctl change_password

5. AMQP operations-to-RabbitMQ permissions map
![](https://upload-images.jianshu.io/upload_images/4108852-53a99e1f61395517.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. An access control entry consists of four parts:
- The user being granted access
- The vhost on which the permissions apply
- The combination of read/write/configure permissions to grant
- The permission scope - whether the permissions apply only to client-named queues/exchange, server-named queues/exchanges, or both. Client-named means your app set the name of the exchange/queue; server-named means your app didn't supply a name and let the server assign a random one for you.

7. RabbitMQ-related operation:
- rabbitmqctl list_queues [-p <VHostPath>] [<QueueInfoItem>...]
Queue information can include 'name', 'messages', 'consumers', 'memory', 'durable', 'auto_delete'
- rabbitmqctl list_exchanges [<ExchangeInfoItem>]
ExchangeInfoItem must be a member of the list [name, type, durable, auto_delete, arguments]. The default is to display name and type.
- rabbitmqctl list_bindings
The output consists of rows containing the exchange name, queue name, routing key, and arguments.

8. The log of RabbitMQ is under  `LOG_BASE` environment variable. It is set to **/var/log/rabbitmq** by default.
Inside that folder RabbitMQ will create two log files: **RABBITMQ_NODENAME-sasl.log** and **RABBITMQ_NODENAME.log**.
When RabbitMQ logs Erlang-related information, it'll go to the rabbit-sasl.log files. For example, there you can find Erlang's crash reports that can be helpful when debugging a RabbitMQ node that doesn't want to start.

9. There is an exchange called **amq.rabbitmq.log** whose type is "topic". RabbitMQ will publish its logs to that exchange using the severity level as a routing key - you'll get *error*, *warning* and *info*. You can build a real-time system based on this point.

10. How *rabbitmqctl* works?
*rabbitmqctl* will start up an Erlang node, and from there it'll try to communicate with the RabbitMQ node using the Erlang distribution system. For this to work, you need two things: the proper `Erlang cookie` and the proper `node name`. So what's an Erlang cookie? An Erlang cookie acts as a secret token that two Erlang nodes exchange in order to authenticate themselves. Since you can execute commands on the remote node once you're connected to it, there's a need to make sure that the peer is trusted.
In order for `rabbitmqctl` to communicate with the RabbitMQ node, it needs to share the same cookie. In production you'll probably want to create a`rabbitmq` user and run the server with that user. This means that you must share the cookie with the `rabbitmq` user or you have to switch to that user to be able to execute `rabbitmqctl` successfully.

#### Clustering and dealing with failure

1. RabbitMQ is keeping track of four kinds of internal metadata:
- Queue metadata: Queue names and their properties care they durable or auto-delete.
- Exchange metadata: The exchange's name, the type of exchange it is, and what the properties are
- Binding metadata: A simple table showing how to route messages to queues
- Vhost metadata: Namespacing and security attributes for the queues, exchanges, and bindings within a vhost.

  With a single node, RabbitMQ stores all of this information in memory, while writing it to disk for any queues and exchanges marked durable. Writing it to disk is what ensures that your queues and exchanges will be re-created when you restart a RabbitMQ node. When you add clustering into the mix, RabbitMQ now has to keep track of a new type of metadata: cluster node location and the nodes' relationships to the other types of metadata already being tracked. Clustering also adds the choice about whether to store metadata on disk or in RAM only.

2. In a cluster when you create queues, the cluster only creates the full information about the queue(metadata, state, contents) on a single node in the cluster rather than on all of them(queues are created on the node to which the client declaring the queue is connected). The result is that only the owner node for a queue knows the full information about that queue. All of the non-owner nodes only know the queue's metadata and a pointer to the node where the queue actually lives. So when a cluster node dies, that node's queues and associated binding disappear. Consumers attached to these queues lose their subscriptions, and any new messages that would've matched that queue's binding become black-holed.

3. Exchanges are a figment of your imagination, they don't have their processes, they are just a name and a list of queue bindings. When you publish a message into an exchange, what really happens is the channel you're connect to compares the routing key on the message to the list of bindings for that exchange, and then route it. Then channel does the actual routing of the message to the queue as specified by the matching binding.

4. Every node in the cluster has all of the information about every exchange. For availablity this is great, because it means you don't have to worry about redeclaring an exchange when a node goes down.

5. The `basic.publish` AMQP command doesn't return the status of message. It means that you risk lossing messages. To solve this problems, try AMQP transaction or publisher confirms.

6. Every RabbitMQ node, whether it's a single node system or a part of a larger cluster, is either a RAM node or a disk node. A RAM node stores all of the metadata defining the queues, exchanges, binding, users, permissions, and vhosts only in RAM, whereas a disk node also saves the metadata to disk.
When you declare a queue, exchange, or binding in a cluster, the operation won't return until all of the cluster nodes have successfully committed the metadata changes. For a RAM node, this means writing the changes into memory, but for a disk node this means an expensive disk write that must complete before the node can say "I've got it."
RabbitMQ only requires that one node in a cluster be a disk node. Every other node can be a RAM node. Keep in mind that when nodes join or leave a cluster, they need to be able to notify at least one disk node of the change. If you only have one disk node and that node happens to be down, your cluster can continue to route messages but you can't do any of the following:
- create queues
- create exchanges
- create bindings
- add users
- change permissions
- add or remove cluster nodes
In other words, your cluster can keep running  if its sole disk node is down but you'll be unable to change anything until you can store that node to the cluster. The solution is to make two disk nodes in your cluster so as least one of them is available to persist metadata changes at any given time. The only operation all of the disk nodes need to be online for is adding or removing cluster nodes. When RAM nodes restart, they connect to the disk nodes they're configured with to download the current copy of the cluster's metadata. If you only tell a RAM node about one of your two disk nodes and that disk node is down when the RAM node's power cable gets knocked loose, the RAM node won't be able to find the cluster when it reboots. So when you join RAM nodes, make sure they're told about all of your disks nodes. As long as the RAM nodecan find at least one disk node, it can restart and happily rejoin the cluster.

7. RabbitMQ clustering is sensitive to latency and should only be used over a local area network. Using it to provide geographic availability/routing over a WAN will cause timeouts and strange cluster behaviour, so it's ill-advised.

#### Mirrored queue

1. Mirrored queues have slave copies on other nodes in the cluster. In the event that the queue's master node becomes unavailable, the oldest slave will be elected as the new master.

2. Declaring a mirrored queue is just like declaring a normal queue; you pass an extra argument called `x-ha-policy` to the "queue.declare" call.

3. If a mirrored queue loses a slave node, any consumers attached to the mirrored queue don't notice the loss. That's because technically they're attached to the queue's master copy. But if the node hosting the master copy fails, all of the queue's consumers need to reattach to start listening to the new queue master. For consumers that were connected through the node that actually failed, this isn't hard. Since  they've lost their TCP onnection to the node, they'll automatically pick up the new queue master when they reattach to a new node in the cluster. But for consumers that wre attached to the mirrored queue through a node that didn't fail, RabbitMQ will send those consumer a `consumer cancellation` notification telling them they've no longer attached to the queue master. If your AMQP client library understands consumer cancellation notifications, it'll raise an exception and your app will know it's no longer attached to the queue and need to reattach. On the other hand, if your client library doesn't understand consumer cancellations, you're in a bind. The client has no way of telling your app that its consumption loops point to a master queue copy that no longer exists. Unfortuntely, there's no clever way around this situation. So if your client library doesn't understand consumer cancellation notifications, you should avoid mirrored queues until it does.

4. When the master node of mirrored queue fails, consumed but unacknowledged messages are required to their original positions in the queue.

#### Lost connections and failing clients between servers

1. Consider what assumptions you can make before writing your code:
- If I reconnect to a new server, what happens to my channels and all of the consumption loops attached to them? They're invalid now and point nowhere. You need to rebuild both.
- When I reconnect, can I assume that all of my exchanges, queues, and bindings are still in the cluster? Can I just reconnect and immediately start consuming from my queues again? No, you can't assume queues and bindings survived the node failure. You must assume that all of the queues you were consuming from were hosted on the node you were aattached to - and no longer exists. If you're using Rabbit's built-in clustering you can assume exchange will survive node failure due to replica to every node, but if you're using an active/standby setup, you can't even assume exchanges will survive failover.

2. You should always treat failover as if you were connecting to a completely unrelated RabbitMQ server, rather than a cluster node with some shared state. As a result, whenever a node failure occurs, the first order of business after detecting the failure and reconnecting is to rebuild the fabric of exchanges, queues, and binding that your app needs to operate.

#### Warrens and shovels: failover and replication

1. Clustering makes you trade the benefit of all the nodes acting as single unit to distribute the load for the drawable of not being able to use durable queues on downed nodes until the nodes are restored. Also, clustering won't give you what you need to build a RabbitMQ architecture that's distributed across more than one data center.

2. Since version 1.8.0, when a node with a durable queue goes down, that queue can't be re-created. Any client that attempts to redeclare the queue will receive a 404 NOT_FOUND AMQP error. Until that node is restored, any messages that would've been delivered to it are either black-holed or errors are sent to the clients that set mandatory publish flags.

3. If your application can't risk losing messages or deal with the latency of continuously republishing messages until therir downed queue returns, then you need what we call warrens. In our parlance, a warren is a pair of active/standby standalone servers with a load balancer in front handling failover.

4. The Warren can't make standby node to have all of the messages that were in the active node when if failed. The approach with load balancers and shared-nothing architecture doesn't give you this. Instead the approach gives you an immediate place to start publishing and consuming messages again, and when you restore the active node, it allows your consumers to reattach and drain the messages that wre in the queues when the active node went down. You don't lose any messages old or new, but you do have to wait for the active node to be restored for the old messages to become available again.

5. The other school of thought for building a warren says you should instead build it with shared storage betwen your active and standby servers with RabbitMQ not running on the standby node. Then when a failure of the active server happens you use Pacemaker to transfer the RabbitMQ IP address to the standby node and then start up Rabbit on that node to pick up your current metadata, contents, and state from the shared storage. There are only a couple of problems with that setup in our opinion. First, the storage is shared, so if some kinds of corruption kills your active node, that corruption will be present on the standby node too and prevent RabbitMQ from starting there. Second, you need to be sure that the standby RabbitMQ has the same node name and UID as RabbitMQ on the primary node. If either of these aren't true, the standby Rabbit won't be able to access the files on the shared storage and fire up. Lastly, using this setup for a warren means your standby Rabbit isn't actually running. So there's a possibility that something you have changed on the standby node that will prevent Rabbit from starting up when you need it.

#### Long-distance communication and replication

1. RabbitMQ has no strategy to cope with network partitioning if that WAN link fails.

2. Shovel is a plugin for RabbitMQ that enables you to define replication relationships betwen a queue on one RabbitMQ server and an exchange on another.

3. Whether you need to replicate messages betwen RabbitMQ servers across the country or across the street, shovel is your go-to solution.
