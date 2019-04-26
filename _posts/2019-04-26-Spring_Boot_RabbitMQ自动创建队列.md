---
layout: post
title: Spring Boot RabbitMQ自动创建队列
date: 2019-04-26
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- RabbitMQ
tags:
- RabbitMQ
---

最近我们在做自动化部署的时候，出现了一只拦路虎，就是我们需要在启动微服务的时候，自动创建RabbitMQ队列。

刚开始我们得知，Consumer会自动创建队列，而实际我们的代码中，并没有自动创建。我们有如下代码:

Listener.java
~~~
@SpringBootApplication
public class Listener {

    public static void main(String[] args) {
        SpringApplication.run(Consumer.class, args);
    }


}
~~~

Consumer.java
~~~
@Component
public class Consumer implements Log{

    @RabbitListener(queues = "test.queue")
    public Integer process(@Headers Map<String, Object> header, @Payload ReckonRequestBO reckonRequestBO) throws IOException {
        return -1;
    }
}
~~~

Configuration.java
~~~
@Configuration
public class Configuration {


    @Value("test.queue")
    private String pandoraQueue;


    private final ErrorHandler errorHandler;


    @Autowired
    public Configuration(ErrorHandler errorHandler) {
        this.errorHandler = errorHandler;
    }

    @Bean
    Queue subscribeQueue(){

        return QueueBuilder
                .durable(pandoraQueue)
                .build();

    }

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            SimpleRabbitListenerContainerFactoryConfigurer configurer,
            ConnectionFactory connectionFactory){

        SimpleRabbitListenerContainerFactory factory=new SimpleRabbitListenerContainerFactory();

        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        factory.setErrorHandler(errorHandler);

        configurer.configure(factory,connectionFactory);

        return factory;
    }
}
~~~

运行这些代码，会发现不能自动创建队列。

由于我们对SpringBoot的启动流程(代码级的)并不清楚，所以开始我们只能靠猜。我们猜测是上述`Configuration.java`中的`rabbitListenerContainerFactory（）`方法写的有问题。

于是查看`SimpleRabbitListenerContainerFactory`的实现,发现其中有这么一个方法:
~~~
@Override
protected void initializeContainer(SimpleMessageListenerContainer instance) {
  super.initializeContainer(instance);

  if (this.applicationContext != null) {
    instance.setApplicationContext(this.applicationContext);
  }
  if (this.taskExecutor != null) {
    instance.setTaskExecutor(this.taskExecutor);
  }
  if (this.transactionManager != null) {
    instance.setTransactionManager(this.transactionManager);
  }
  if (this.txSize != null) {
    instance.setTxSize(this.txSize);
  }
  if (this.concurrentConsumers != null) {
    instance.setConcurrentConsumers(this.concurrentConsumers);
  }
  if (this.maxConcurrentConsumers != null) {
    instance.setMaxConcurrentConsumers(this.maxConcurrentConsumers);
  }
  if (this.startConsumerMinInterval != null) {
    instance.setStartConsumerMinInterval(this.startConsumerMinInterval);
  }
  if (this.stopConsumerMinInterval != null) {
    instance.setStopConsumerMinInterval(this.stopConsumerMinInterval);
  }
  if (this.consecutiveActiveTrigger != null) {
    instance.setConsecutiveActiveTrigger(this.consecutiveActiveTrigger);
  }
  if (this.consecutiveIdleTrigger != null) {
    instance.setConsecutiveIdleTrigger(this.consecutiveIdleTrigger);
  }
  if (this.prefetchCount != null) {
    instance.setPrefetchCount(this.prefetchCount);
  }
  if (this.receiveTimeout != null) {
    instance.setReceiveTimeout(this.receiveTimeout);
  }
  if (this.defaultRequeueRejected != null) {
    instance.setDefaultRequeueRejected(this.defaultRequeueRejected);
  }
  if (this.adviceChain != null) {
    instance.setAdviceChain(this.adviceChain);
  }
  if (this.recoveryBackOff != null) {
    instance.setRecoveryBackOff(this.recoveryBackOff);
  }
  if (this.mismatchedQueuesFatal != null) {
    instance.setMismatchedQueuesFatal(this.mismatchedQueuesFatal);
  }
  if (this.missingQueuesFatal != null) {
    instance.setMissingQueuesFatal(this.missingQueuesFatal);
  }
  if (this.consumerTagStrategy != null) {
    instance.setConsumerTagStrategy(this.consumerTagStrategy);
  }
  if (this.idleEventInterval != null) {
    instance.setIdleEventInterval(this.idleEventInterval);
  }
  if (this.applicationEventPublisher != null) {
    instance.setApplicationEventPublisher(this.applicationEventPublisher);
  }
}
~~~

在`SimpleRabbitListenerContainerFactory`类中，没有发现任何可能跟创建队列有关的配置项，看到上面方法又配置了`SimpleMessageListenerContainer`，于是进一步看`SimpleMessageListenerContainer`这个类的实现。

在`SimpleMessageListenerContainer`类中，我们发现了这么一个属性:
~~~
/**
 * @param autoDeclare the boolean flag to indicate an redeclaration operation.
 * @since 1.4
 * @see #redeclareElementsIfNecessary
 */
public void setAutoDeclare(boolean autoDeclare) {
  this.autoDeclare = autoDeclare;
}
~~~

可以看到，这个应该就是跟自动创建队列相关的。然后我们通过查看调用，发现它在`SimpleMessageListenerContainer.doStart()`中被调用，代码如下:
~~~
/**
	 * Re-initializes this container's Rabbit message consumers, if not initialized already. Then submits each consumer
	 * to this container's task executor.
	 * @throws Exception Any Exception.
	 */
	@Override
	protected void doStart() throws Exception {
    ....
		checkMismatchedQueues();
		....
	}
~~~

上面省略了很多无关代码，我们点进去看`checkMismatchedQueues()`方法的实现，如下:
~~~
private void checkMismatchedQueues() {
  if (this.mismatchedQueuesFatal && this.rabbitAdmin != null) {
    try {
      this.rabbitAdmin.initialize();
    }
    catch (AmqpConnectException e) {
      logger.info("Broker not available; cannot check queue declarations");
    }
    catch (AmqpIOException e) {
      if (RabbitUtils.isMismatchedQueueArgs(e)) {
        throw new FatalListenerStartupException("Mismatched queues", e);
      }
      else {
        logger.info("Failed to get connection during start(): " + e);
      }
    }
  }
}
~~~

这里需要注意的是，只有当`SimpleMessageListenerContainer.mismatchedQueuesFatal`这个属性是true的时候，才会调用后面的方法。

然后读下去，会发现最终调用的是`RabbitAdmin.initialize()`方法，而其中包含了下面的片段:
~~~
/**
 * Declares all the exchanges, queues and bindings in the enclosing application context, if any. It should be safe
 * (but unnecessary) to call this method more than once.
 */
public void initialize() {
  ....
  this.rabbitTemplate.execute(new ChannelCallback<Object>() {
    @Override
    public Object doInRabbit(Channel channel) throws Exception {
      declareExchanges(channel, exchanges.toArray(new Exchange[exchanges.size()]));
      declareQueues(channel, queues.toArray(new Queue[queues.size()]));
      declareBindings(channel, bindings.toArray(new Binding[bindings.size()]));
      return null;
    }
  });
  this.logger.debug("Declarations finished");

}
~~~

很明显其中有创建队列的代码，如下:
~~~
private DeclareOk[] declareQueues(final Channel channel, final Queue... queues) throws IOException {
		List<DeclareOk> declareOks = new ArrayList<DeclareOk>(queues.length);
		for (int i = 0; i < queues.length; i++) {
			Queue queue = queues[i];
			if (!queue.getName().startsWith("amq.")) {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("declaring Queue '" + queue.getName() + "'");
				}
				try {
					try {
						DeclareOk declareOk = channel.queueDeclare(queue.getName(), queue.isDurable(),
								queue.isExclusive(), queue.isAutoDelete(), queue.getArguments());
						declareOks.add(declareOk);
					}
					catch (IllegalArgumentException e) {
						if (this.logger.isDebugEnabled()) {
							this.logger.error("Exception while declaring queue: '" + queue.getName() + "'");
						}
						try {
							if (channel instanceof ChannelProxy) {
								((ChannelProxy) channel).getTargetChannel().close();
							}
						}
						catch (TimeoutException e1) {
						}
						throw new IOException(e);
					}
				}
				catch (IOException e) {
					logOrRethrowDeclarationException(queue, "queue", e);
				}
			}
			else if (this.logger.isDebugEnabled()) {
				this.logger.debug(queue.getName() + ": Queue with name that starts with 'amq.' cannot be declared.");
			}
		}
		return declareOks.toArray(new DeclareOk[declareOks.size()]);
	}
~~~

代码很简单，就不解释了。

总结一下，我们发现，要想自动创建队列，`SimpleMessageListenerContainer`需要满足这么两点:
- mismatchedQueuesFatal属性设置为true
- autoDeclare属性也设置为true

而配合上面的`SimpleRabbitListenerContainerFactory.initializeContainer(SimpleMessageListenerContainer instance)`会对SimpleMessageListenerContainer进行配置,我们就可以提出这种方案:

写一个`CustomRabbitListenerContainerFactory`，继承`SimpleRabbitListenerContainerFactory`,代码如下:
~~~
public class CustomRabbitListenerContainerFactory extends SimpleRabbitListenerContainerFactory {

    @Override
    protected void initializeContainer(SimpleMessageListenerContainer instance) {
        super.initializeContainer(instance);
        instance.setAutoDeclare(true);
        instance.setMismatchedQueuesFatal(true);
    }

}
~~~

然后在`Configuration.java`中使用这个`ContainerFactory`:
~~~
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        SimpleRabbitListenerContainerFactoryConfigurer configurer,
        ConnectionFactory connectionFactory){

    CustomRabbitListenerContainerFactory factory = new CustomRabbitListenerContainerFactory();

    factory.setMessageConverter(new Jackson2JsonMessageConverter());
    factory.setErrorHandler(errorHandler);

    configurer.configure(factory,connectionFactory);

    return factory;
}
~~~

这样子就可以解决这个问题啦。
