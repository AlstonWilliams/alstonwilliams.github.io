---
layout: post
title: 《Netty-in-Action》读书笔记
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
Chapter2
1. SimpleChannelInboundHandler vs ChannelInboundHandler:
In the client, when `channelRead0()` completes, you have the incoming message and you're done with it. When the method returns, `SimpleChannelInboundHandler` takes care of releasing the memory reference to the `ByteBuf` that holds the message. But `ChannelInboundHandler` doesn't release the message at the point.



Chapter3
1. Channel-sockets
	EventLoop - Control flow, multithreading, concurrency
	ChannelFuture - Asynchronous notification
2. Channel's implementation:
- EmbeddedChannel
- LocalServerChannel
- NioDatagramChannel
- NioSctpChannel
- NioSocketChannel
3. `EventLoop` defines Netty's core abstraction for handling events that occur during the lifetime of a connection.
4. The relationship between `Channel`, `EventLoop`, `Thread`, and `EventLoopGroup` are:
- An `EventLoopGroup` contains one or more `EventLoop`s
- An `EventLoop` is bound to a single Thread for its lifetime
- All I/O events processed by an `EventLoop` are handled on its dedicated `Thread`
- A `Channel` is registered for its lifetime with a single `EventLoop`
- A single `EventLoop` may be assigned to one or more `Channel`s
4. Because all I/O operations in Netty are asynchronous, so we need a way to determine its result at a later time. So Netty provides `ChannelFuture`, whose `addListener()` method registers a `ChannelFutureListener` to be notified when an operation has completed.
5. `ChannelHandler` serves as the container for all application logic that applies to handling inbound and outbound data. This is possible because `ChannelHandler` methods are triggered by network events. 
6. `ChannelPipeline` provides a container for a chain of `ChannelHandler`s and defines an API for propagating the flow of inbound and outbound events along the chain. When a `Channel` is created, it is automatically assigned its own `ChannelPipeline`.
7. `ChannelHandler`s are installed in the `ChannelPipeline` as follows:
- A `ChannelInitializer` implementation is registered with a `ServerBootstrap`
- When `ChannelInitializer.initChannel()` is called, the `ChannelInitializer` installs a custom set of `ChannelHandler`s in the pipeline. 
- The `ChannelInitializer` removes itself from the `ChannelPipeline`
8. If a message or any other inbound event is read, it will start from the head of the pipeline and be passed to the first `ChannelInboundHandler`. But data flows from the tail through the chain of `ChannelOutboundHandlers` until it reaches the head. 
9. There are two ways of sending messages in Netty. You can write directly to the `Channel` or write to a `ChannelHandlerContext` object associated with a `ChannelHandler`. The former approach causes the message to start from the tail of the `ChannelPipeline`, the latter causes the message to start from the next handler in the `ChannelPipeline`.
10. Adapters you'll call most often when creating your custom handlers:
- ChannelHandlerAdapter
- ChannelInboundHandlerAdapter
- ChannelOutboundHandlerAdapter
- ChannelDuplexHandlerAdapter
11. Why bootstrapping a client requires only a single `EventLoopGroup`, but a `ServerBootstrap` requires two(which can be the same instance)?
A server needs two distinct sets of `Channel`s. The first set will contain a single `ServerChannel` representing the server's own listening socket, bound to a local port. The second set will contain all the `Channel`s that have been created to handle incoming client connections - one for each connection the server has accepted.



Chapter4
1. The implementation of `compareTo()` in `AbstractChannel` throws an `Error` if two distinct `Channel` instances return the same hash code.
2. Typical uses for `ChannelHandler`s include:
- Transforming data from one format to another
- Providing notification of exceptions
- Providing notification of a `Channel` becoming active or inactive
- Providing notification when a `Channel` is registered with or deregistered from an `EventLoop`
- Providing notification about user-defined events
3. Netty's `Channel` implementations are thread-safe, so you can store a reference to a `Channel` and use it whenever you need to write something to the remote peer, even when many threads are in use.
4. Netty-provided transports:
- NIO: io.netty.channel.socket.nio Uses the `java.nio.channels` package as a foundation - a selector-based approach
- Epoll: io.netty.channel.epoll Uses JNI for `epoll()` and non-blocking IO. This transport supports features available only on Linux, such as `SO_REUSEPORT`, and is faster than the NIO transport as well as fully non-blocking
- OIO: io.netty.channel.socket.oio Uses the `java.net` package as a foundation - uses blocking streams.
- Local: io.netty.channel.local A local transport that can be used to communicate in the VM via pipes
- Embedded: io.netty.channel.embedded An embedded transport, which allows using `ChannelHandlers` without a true network-based transport. This can be quite useful for testing your ChannelHandler implementations.

Chapter5
1. Netty's API for data handling is exposed through two components - abstract class ByteBuf and interface ByteBufHolder.
These are some of the advantages of the `ByteBuf` API:
- It's extensible for user-defined buffer types
- Transparent zero-copy is achieved by a built-in composite buffer type
- Capacity is expanded on demand
- Switching between reader and writer modes doesn't require calling ByteBuffer's flip() method
- Reading and writing employ distinct indices
- Method chaining is supported
- Reference counting is supported
- Pooling is supported
2. How `ByteBuf` works?
ByteBuf maintains two distinct indices: one for reading and one for writing. When you read from `ByteBuf`, its `readerIndex` is incremented by the number of bytes read. Similarly, when you write to `ByteBuf`, its `writerIndex` is incremented.
3. `ByteBuf` methods whose name begins with `read` or `write` advance the corresponding index, whereas operations that begins with `set` or `get` do not. The latter methods operate on a relative index that's passed as an argument to the method.
4. `ByteBuf` usage pattern:
- Heap buffers: Store the data in the JVM heap as an array.
- DIRECT BUFFER: The performance is better because it avoids copy data from JVM to DIRECT BUFFER. But it is difficult to allocate or release direct buffer.
- COMPOSITE BUFFER: Netty implements this pattern with a subclass of `ByteBuf`, `CompositeByteBuf`, which provides a virtual representation of multiple buffers as a single, merged buffer. `CompsiteByteBuf` may not allow access to a backing array, so accessing the data in a `CompositeByteBuf` resembles the direct buffer pattern.
5. The JDK's `InputStream` defines the methods `mark(int readlimit)` and `reset()`. These are used to mark the current position in the stream to a specified value and to reset the stream to that position, respectively.
Similarly, you can set and reposition the `ByteBuf readerIndex` and `ByteBuf writerIndex` by calling `markReaderIndex()`, `markWriterBuffer()`, `resetReaderIndex()`, and `resetWriterIndex()`. These are similar to the `InputStream` calls, expect that there's no `readLimit` to specify when the mark becomes invalid.
6. A `derived buffer` provides a view of a `ByteBuffer` that represents its contents in a specified way. Such views are cerated by the following methods:
- duplicate()
- slice()
- slice(int, int)
- Unpooled.unmodifiableBuffer(...)
- order(ByteOrder)
- readSlice(int)
Each returns a new `ByteBuf` instance with its own reader, writer, and marker indices. The internal storage is shared, so be carefully if you modify its content you are modifying the source instance as well.
7. `ByteBufHolder` is a good choice if you want to implement a message object that stores its payload in a `ByteBuf`.
8. You can obtain a reference to a `ByteBufAllocator` either from a `Channel` or through the `ChannelHandlerContext` that is bound to a `ChannelHandler`. The following listing illustrates both of these methods.
9. Netty provides two implementations of `ByteBufAllocator`: `PooledByteBufAllocator` and `UnpooledByteBufAllocator`. The former pools `ByteBuf` instances to improve performance and minimum memory fragmentation. This implementation uses an efficient approach to memory allocation known as `jemalloc` that has been adopted by a number of modern OSes. The latter implementation doesn't pool `ByteBuf` instances and returns a new instance everytime it is called.


Chapter6
1. Channel lifecycle states:
- ChannelUnregistered: The Channel was created, but isn't registered to an `EventLoop`
- ChannelRegistered: The channel is registered to an `EventLoop`
- ChannelActive: The Channel is active(connected to its remote peer). It's now possible to receive and send data.
- ChannelInactive: The Channel is not connected to the remote peer
2. ChannelHandler lifecycle methods:
- handlerAdded: Called when a ChannelHandler is added to a ChannelPipeline
- handlerRemoved: Called when a ChannelHandler is removed from a ChannelPipeline
- exceptionCaught: Called if an error occurs in the ChannelPipeline during processing
3. ChannelHandler's subinterface:
- ChannelInboundHandler
- ChannelOutboundHandler
4. ChannelInboundHandler methods:
- channelRegistered
- channelUnregistered
- channelActive
- channelInactive
- channelReadComplete
- channelRead
- channelWritabilityChanged: Invoked when the writablity state of the Channel changes. The user can ensure writes are not done too quickly or can resume writes when the Channel becomes writable again. The Channel method isWritable() can be called to detect the writability of the channel. The threshold for writability can be set via `Channel.config().setWriteHighWaterMark()` and `Channel.config().setWriteLowWaterMark()`
- userEventTriggered: Invoked when ChannelInboundHandler.fireUserEventTriggered() is called because a POJO was passed through the ChannelPipeline.
5. ChannelOutboundHandler:
- bind(ChannelHandlerContext, SocketAddress, ChannelPromise)
- connect(ChannelHandlerContext, SocketAddress, SocketAddress, ChannelPromise)
- disconnect(ChannelHandlerContext, ChannelPromise)
- close(ChannelHandlerContext, ChannelPromise)
- deregister(ChannelHandlerContext, ChannelPromise)
- read(ChannelHandlerContext)
- write(ChannelHandlerContext, Object, ChannelPromise)
- flush(ChannelHandlerContext) 
6. To assist you in diagnosing potential problems, Netty provides `ResourceLeakDetector`, which will sample about 1% of your application's buffer allocations to check for memory leaks. The overhead involved is very small.
7. Leak-detection levels:
- DISABLED: Disables leak detection. Use this only after extensive testing
- SIMPLE: Reports any leaks found using the default sampling rate of 1%. This is the default level and is a good fit for most cases.
- ADVANCED: Reports leaks found and where the message was accessed. Uses the default sampling rate.
- PARANOID: Like ADVANCED except that every access is sampled. This has a heavy impact on performance and should be used only in the debugging phase.
The leak-detection level is defined by this one:
java -Dio.netty.leakDetectionLevel=ADVANCED
8. Every new `Channel` that's created is assigned a new `ChannelPipeline`. This association is permanent; the Channel can neither attach another `ChannelPipeline` nor detach the current one. This is a fixed operation in Netty's component lifecycle and requires no action on the part of the developer.
9. The `ChannelHandlerContext` associated with a `ChannelHandler` never changes, so it's safe to cache a reference to it.
`ChannelHandlerContext` methods, involve a shorter event flow than do the identically named methods available on other classes. This should be exploited where possible to provide maximum performance.
10. Use `@Sharable` only if you're certain that your ChannelHandler is thread-safe.
11. Because the exception will continue to flow in the inbound direction, the `ChannelInboundHandler` that implements the preceding logic is usually placed last in the `ChannelPipeline`. This ensures that all inbound exceptions are always handled, wherever in the `ChannelPipeline` they may occur.



Chapter7
1. I/O operations in Netty3:
The threading model used in previous releases guaranteed only that inbound events would be executed in the so-called I/O thread. All outbound events were handled by the calling thread, which might be the I/O thread or any other. This seemed a good idea at first but was found to be problematical because of the need for careful synchronization of outbound events in ChannelHandlers. In shorter, it wasn't possible to guarantee that multiple thread wouldn't try to access an outbound event at the same time. This could happen, for example, if you fired simultaneous downstream events for the same Channel by calling Channel.write() in different threads.
The threading model adopted in Netty4 resolves these problems by handling everything that occurs in a given `EventLoop` in the same thread. This provides a simpler execution architecture and eliminates the need for synchronization in the `ChannelHandler`s.
2. The `EventLoop`s that service I/O and events for `Channel`s are contained in an `EventLoopGroup`. The manner in which `EventLoop`s are created and assigned varies according to the transport implementation.
- Asynchronous transports: Asynchronous implementations use only a few `EventLoop`s (and their associated Threads), and in the current model these may be shared among `Channel`s. This allows many `Channel`s to be served by the smallest possible number of `Thread`s, rather than assigning a `Thread` per `Channel`. Be aware of the implications of `EventLoop` allocation for `ThreadLocal` use. Because an `EventLoop` usually powers more than one `Channel`, `ThreadLocal` will be the same for all associated `Channel`s. This makes it a poor choice for implementing a function such as state tracking. However, in a stateless context it can still be useful for sharing heavy or expensive objects, or even events, among `Channel`s.
- Blocking transports: One `EventLoop` (and its Thread) is assigned to each `Channel`. You may have encountered this pattern if you've developed applications that use the blocking I/O implementation in the `java.io` package.



Chapter8
1. The differences between `handler()` and `childHandler()` is that the former adds a handler that's processed by the accepting `ServerChannel`, whereas `childHandler()` adds a handler that's processed by an accepted `Channel`, which represents a socket bound to a remote peer.
2. Reuse `EventLoop`s wherever possible to reduce the cost of thread creation.


Chapter11
1. Provided ChannelHandlers and codec:
- SslHandler
- HTTP decoders and encoders:
	- HttpRequestEncoder: Encodes `HttpRequest`, `HttpContent`, and `LastHttpContent` messages to bytes
	- HttpResponseEncoder: Encodes `HttpResponse`, `HttpContent`, and `LastHttpContent` messages to bytes
	- HttpRequestDecoder: Decodes `bytes` into `HttpRequest`, `HttpContent`, and `LastHttpContent` message.
	- HttpResponseDecoder: Decodes `bytes` into `HttpResponse`, `HttpContent`, and `LastHttpContent` message
	- HttpClientCodec: Package `HttpRequestEncoder` and `HttpResponseDecoder`
	- HttpServerCodec: Package `HttpRequestDecoder` and `HttpResponseEncoder`
- HttpObjectAggregator: Aggregate multiple HTTPObject into `FullHttpRequest` and `FullHttpResponse`
- HttpContentCompressor: Compress HTTP content, support `gzip` and `deflate` now
- HttpContentDecompressor: Decompress HTTP content
- IdleStateHandler: Fires an `IdleStateEvent` if the connection idle too long. You can then handle the `IdleStateEvent` by overriding `userEventTriggered()` in your `ChannelInboundHandler`.
- ReadTimeoutHandler: Throws a `ReadTimeoutException` and closes the `Channel` when no inbound data is received for a specified interval. The `ReadTimeoutException` can be detected by overriding `exceptionCaught()` in your `ChannelHandler`
- WriteTimeoutHandler: Throws a `WriteTimeoutException` and closes the `Channel` when no inbound data is received for a specified interval. The `WriteTimeoutException` can be detected by overriding `exceptionCaught()` in your `ChannelHandler`.
- DelimiterBasedFrameDecoder: A generic decoder that extracts frames using any user-provided delimiter
- LineBasedFrameDecoder: A decoder that extracts frames delimited by the line-endings `
` or `
`. This decoder is faster than `DelimiterBasedFrameDecoder`
2. ChunkedInput implementations:
- ChunkedFile: Fetches data from a file chunked by chunk, for use when your platform doesn't support zero-copy or you need to transform the data
- ChunkedNioFile: Similar to `ChunkedFile` expect that it uses `FileChannel`
- ChunkedStream: Transfers content chunk by chunk from an `InputStream`
- ChunkedNioStream: Transfers content chunk by chunk from a `ReadableByteChannel`
3. To use your own `ChunkedInput` implementation install a `ChunkedWriteHandler` in the pipeline. Use `ChunkedWriteHandler` to write large data without risking `OutOfMemoryErrors`.
4. JDK serialization codecs:
- CompatibleObjectDecoder: Decoder for interoperating with non-Netty peers that use JDK serialization
- CompatibleObjectEncoder: Encoder for interoperating with non-Netty peers that use JDK serialization
- ObjectDecoder: Decoder that uses custom serialization for decoding on top of JDK serialization; it provides a speed improvement when external dependencies are excluded. Otherwise the other serialization implementations are preferable.
- ObjectEncoder: Encoder that uses custom serialization for decoding on top of JDK serialization; it provides a speed improvement when external dependencies are excluded. Otherwise the other serialization implementations are preferable.
5. JBoss marshalling. If you are free to make use of external dependencies, JBoss marshalling is ideal: It's up to 3 times faster than JDK serialization and more compact.
6. JBoss marshalling codecs:
- CompatibleMarshallingDecoder: For compatibility with peers that use JDK serialization
- CompatibleMarshallingEncoder
- MarshallingDecoder: For use with peers that use JBoss Marshalling. These classes must be used together.
- MarshallingEncoder
7. Protobuf codec:
- ProtobufDecoder: Decodes a message using Protobuf
- ProtobufEncoder: Encodes a message using Protobuf
- ProtobufVarint32FrameDecoder: Splits received `ByteBuf`s dynamically by the value of the Google Protocol "Bse 128 Varints" integer length field in the message
