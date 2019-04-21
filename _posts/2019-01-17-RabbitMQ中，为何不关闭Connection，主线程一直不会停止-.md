---
layout: post
title: RabbitMQ中，为何不关闭Connection，主线程一直不会停止
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- RabbitMQ
tags:
- RabbitMQ
---
在Spark中，我们在Driver里放了一个RabbitMQ，接收消息，然后处理。避免一次次地上传Jar包，进行前面的准备操作。

然而，在实际操作中，发现了一些诡异的事情。

第一件事情就是，当我们使用`yarn -kill`命令，将这个Job kill掉之后，我们发现，实际上Driver并没有被kill掉。还是会继续监听那个队列，而且，`SparkContext`确实是被关闭掉了。

而正常的情况，应当是`yarn -kill`以后，Driver被kill掉。

于是，我猜想，是因为在poll消息的时候，由于有**wait**存在，使Driver的线程阻塞了，不能接收到ApplicationMaster发送来的kill消息，导致Driver不会停掉。

然而，在验证这个想法的过程中，我发现，即使不poll消息，Driver还是不会停止。

代码如下:
~~~
package com.hyper.cdp.label.utils;

import com.hyper.util.CacheConfUtils;
import com.hyper.util.Log;
import com.hyper.util.Mapper;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.QueueingConsumer;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitMQAccessor implements Log {

    private String host;
    private String port;
    private String username;
    private String password;
    private String vhost;
    private String queue;

    private ConnectionFactory connectionFactory;
    private Connection connection;
    private Channel channel;
    private QueueingConsumer queueingConsumer;

    public RabbitMQAccessor(String queue) {
        this(CacheConfUtils.get("amqp.host", ""),
                CacheConfUtils.get("amqp.port", "5672"),
                CacheConfUtils.get("amqp.username", ""),
                CacheConfUtils.get("amqp.password", ""),
                CacheConfUtils.get("amqp.vhost", ""),
                queue);
    }

    public RabbitMQAccessor(String host, String port, String username, String password,
                            String vhost, String queue) {
        this.host = host;
        this.port = port;
        this.username = username;
        this.password = password;
        this.vhost = vhost;
        this.queue = queue;

    }

    public void init() throws IOException, TimeoutException {
        connectionFactory = new ConnectionFactory();
        connectionFactory.setHost(host);
        connectionFactory.setPort(Integer.valueOf(port));
        connectionFactory.setUsername(username);
        connectionFactory.setPassword(password);
        connectionFactory.setVirtualHost(vhost);

        connection = connectionFactory.newConnection();
        channel = connection.createChannel();

        channel.queueDeclare(queue, true, false, false, null);
        queueingConsumer = new QueueingConsumer(channel);
        channel.basicConsume(queue, queueingConsumer);
    }

    private QueueingConsumer.Delivery currentDelivery = null;

    public String poll() {
        String result = null;
        try {
            currentDelivery = queueingConsumer.nextDelivery();
            if (currentDelivery != null) {
                byte[] bs = currentDelivery.getBody();
                result = new String(bs);
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return result;
    }

    public Object pollJson(Class targetClass) {
        Object result = null;

        String string = poll();
        if (string == null || string.trim().equals("")) return null;

        try {
            result = Mapper.readValue(string, targetClass);
        } catch (IOException e) {
            e.printStackTrace();
        }

        return result;
    }

    public void ack() {
        try {
            channel.basicAck(currentDelivery.getEnvelope().getDeliveryTag(), false);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void close() {
        try {
            channel.close();
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws IOException, TimeoutException {
        RabbitMQAccessor rabbitMQAccessor = new RabbitMQAccessor("host",
                "5672",
                "a",
                "b",
                "/c",
                "queue");
        rabbitMQAccessor.init();
        System.out.println("------- before close");
        rabbitMQAccessor.close();

        System.out.println("------- after close");
    }
}
~~~

就简单地初始化了一下，然后关闭一下，想结束掉这个程序，但是它竟然不给我停。

所以我就很奇怪。于是猜测是创建Connection时搞得鬼。

通过断点调试，还真发现是它在捣蛋。

在`AMQConnection`这个类中，我们能看到，在创建Connection时，有这么一段:

![](https://upload-images.jianshu.io/upload_images/4108852-08fcfde266e7863c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们再看`MainLoop`的实现:

![](https://upload-images.jianshu.io/upload_images/4108852-f272296b3270ed9b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最重要的是那个while循环。

我们可以看到，只要`_running`是true，它就会一直运行。而`_running`只有当Connection关闭时，才会是false.

你可能会说，这个是子线程啊。跟主线程不结束有什么关系。

我们从[StackOverflow](https://stackoverflow.com/questions/9651842/will-main-thread-exit-before-child-threads-complete-execution)中可以得知，**在Java中，只要存在一个不是daemon的子线程还在运行，主线程就不会退出**。

![](https://upload-images.jianshu.io/upload_images/4108852-6ec6f957948a9456.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实际上，这个问题很简单。但是一年没碰这种Java的基础知识，好多内容都有点忘记了。所以啊，还是要温故知新啊。

另外，上面对Spark的那个问题，我的猜测是错误的。

上面的代码，正确的应该是，在`close`方法中，加上*connection.close()*
