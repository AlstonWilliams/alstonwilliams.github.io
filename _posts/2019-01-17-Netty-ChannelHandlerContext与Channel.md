---
layout: post
title: Netty-ChannelHandlerContext与Channel
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
当我们在一个Handler中时，我们要想写数据，有两种方式。一种是调用ChannelHandlerContext的相关的写数据的方法，一种是调用ChannelHandlerContext.channel()的相关的写数据的方法。

这两种方法，差别不小。如果不注意，就会踩坑。

我们有如下代码:
~~~
package com.hypers.TestNetty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.concurrent.Future;
import io.netty.util.concurrent.GenericFutureListener;

import java.nio.charset.Charset;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class NettyServerForString {

    private static BlockingQueue<ChannelHandlerContext> queue = new LinkedBlockingQueue<ChannelHandlerContext>();

    public static void main(String[] args) {

        new Thread(new Runnable() {
            @Override
            public void run() {

                ChannelHandlerContext ctx = null;
                while (true) {
                    try {
                        if ((ctx = queue.take()) != null) {
                            String str = "Response " + System.currentTimeMillis();
                            byte[] bytes = str.getBytes(Charset.forName("utf-8"));
                            ByteBuf byteBuf = ctx.alloc().buffer();
                            byteBuf.writeBytes(bytes);
                            ctx.writeAndFlush(byteBuf);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        ServerBootstrap serverBootstrap = new ServerBootstrap();

        NioEventLoopGroup boss = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();

        serverBootstrap
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
                                System.out.println("------ Received: " + msg);
                                queue.put(ctx);
                            }
                        });
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });
        bind(serverBootstrap, 8000);
    }

    private static void bind(final ServerBootstrap serverBootstrap, final int port) {
        serverBootstrap.bind(port).addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                if (!future.isSuccess()) {
                    System.out.println("Try to bind " + (port + 1) + "...");
                    bind(serverBootstrap, port + 1);
                }
            }
        });
    }

}

~~~

这段代码会接收客户端的消息，并给它一个回应。

看起来很简单，其实暗流涌动。

我们可以看到，上面给客户端回消息的时候，我们写的是`ByteBuf`。但是我只是想回一个字符串而已，那我改成这样子好不好?

~~~
package com.hypers.TestNetty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.concurrent.Future;
import io.netty.util.concurrent.GenericFutureListener;

import java.nio.charset.Charset;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class NettyServerForString {

    private static BlockingQueue<ChannelHandlerContext> queue = new LinkedBlockingQueue<ChannelHandlerContext>();

    public static void main(String[] args) {

        new Thread(new Runnable() {
            @Override
            public void run() {

                ChannelHandlerContext ctx = null;
                while (true) {
                    try {
                        if ((ctx = queue.take()) != null) {
                            String str = "Response " + System.currentTimeMillis();                                
                            ctx.writeAndFlush(str);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        ServerBootstrap serverBootstrap = new ServerBootstrap();

        NioEventLoopGroup boss = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();

        serverBootstrap
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
                                System.out.println("------ Received: " + msg);
                                queue.put(ctx);
                            }
                        });
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });
        bind(serverBootstrap, 8000);
    }

    private static void bind(final ServerBootstrap serverBootstrap, final int port) {
        serverBootstrap.bind(port).addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                if (!future.isSuccess()) {
                    System.out.println("Try to bind " + (port + 1) + "...");
                    bind(serverBootstrap, port + 1);
                }
            }
        });
    }

}

~~~

不好意思，不好。

不仅不好，你还会连不好的原因都不知道。不会给你报异常，这条消息就会被静悄悄的扔掉了。

我在调试这个问题的时候，不知道到底是服务器端没有给客户端发送消息，还是客户端收到了以后，由于不是不对，而给扔掉了，最后通过Wireshark抓包，才确定了是服务器端压根就没有给客户端发送消息。

嗯，知道了是服务器端压根就没有给客户端发送消息以后，通过打断点一点点调试，我们会发现错误其实是`java.lang.UnsupportedOperationException: unsupported message type: String (expected: ByteBuf, FileRegion)
`

咦？丫的。我们下面定义的handler的顺序明明是**StringDecoder -> SimpleChannelInboundHandler -> StringEncoder**。那我拿到的是`SimpleChannelInboundHandler`的`ChannelHandlerContext`，那发送消息的时候，接下来不久应该是`StringEncoder`进行处理了么？而且它不就是把String转换成ByteBuf么？有错么？

有错。为啥？

因为`ChannelHandlerContext.writeAndFlush()`在写数据时，实际上，会从后往前(从当前位置)寻找第一个OutboundHandler，然后开始输出。在上面的这个例子里，就是从`SimpleChannelInboundHandler`开始，从后往前找`OutputboundHandler`。而它前面并没有`OutputboundHandler`。所以最后就找到了`DefaultChannelPipeline.HeadContext`。这个是Netty中`Channel`的`Pipeline`的第一个Handler。

然后，将String传递给它，让它进行处理，就会报上面那个错误。

Netty中相关代码如下:

![](https://upload-images.jianshu.io/upload_images/4108852-0070bc2c609cb2c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4108852-997e27a8b7752779.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么，如何改正呢？只需要将`StringEncoder`放到`SimpleChannelInboundHandler`前面就可以了。

但是这样感觉有点奇怪，对吧? `StringEncoder`明明就应该在`SimpleChannelInboundHandler`后面，凭啥提到前面去。

还有一种不需要更改顺序也能跑通的方式，就是使用`ChannelHandlerContext.channel()`的写方法。

~~~
package com.hypers.TestNetty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.concurrent.Future;
import io.netty.util.concurrent.GenericFutureListener;

import java.nio.charset.Charset;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class NettyServerForString {

    private static BlockingQueue<ChannelHandlerContext> queue = new LinkedBlockingQueue<ChannelHandlerContext>();

    public static void main(String[] args) {

        new Thread(new Runnable() {
            @Override
            public void run() {

                ChannelHandlerContext ctx = null;
                while (true) {
                    try {
                        if ((ctx = queue.take()) != null) {
                            String str = "Response " + System.currentTimeMillis();                                
                            ctx.channel().writeAndFlush(str);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        ServerBootstrap serverBootstrap = new ServerBootstrap();

        NioEventLoopGroup boss = new NioEventLoopGroup();
        NioEventLoopGroup worker = new NioEventLoopGroup();

        serverBootstrap
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 1024)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new SimpleChannelInboundHandler<String>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
                                System.out.println("------ Received: " + msg);
                                queue.put(ctx);
                            }
                        });
                        ch.pipeline().addLast(new StringEncoder());
                    }
                });
        bind(serverBootstrap, 8000);
    }

    private static void bind(final ServerBootstrap serverBootstrap, final int port) {
        serverBootstrap.bind(port).addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                if (!future.isSuccess()) {
                    System.out.println("Try to bind " + (port + 1) + "...");
                    bind(serverBootstrap, port + 1);
                }
            }
        });
    }

}

~~~

这样子也可以正确输出。因为`ChannelHandlerContext.channel()`会获取到一个`Channel`，而`Channel`的`writeAndFlush()`方法的定义如下:

![](https://upload-images.jianshu.io/upload_images/4108852-02833098ca1ccb69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`tail`是固定的，就是`TailContext`。它是`Pipeline`中的最后一个`Handler`。然后，它也会从后往前寻找第一个`OutputboundHandler`，找到了`StringEncoder`，然后将String传给它。这样就能给客户端正确的发送消息。

如果你把最后的那个`StringEncoder`去掉，那么还是会出现跟上面类似的错误，导致不能给客户端发送消息。

短短的几行代码，需要注意这么多事情。也是有点醉。
