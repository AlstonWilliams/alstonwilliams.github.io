---
layout: post
title: YARN源码解析(10)-AuxliaryService
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---
这篇文章，实际上并不是源码解析，而是翻译的一篇文章。

在我读过这部分的代码之后，看到了这么一篇文章。介绍的也不错，于是，就直接翻译过来了。

**AuxiliaryService**是NodeManager上的一个Service。它用**yarn.nodemanager.aux-services**来定义。默认值是*mapreduce_shuffle*，比如，在MR2中，是**ShuffleHandler**。这就是你为什么总是从NodeManager的log中看到如下输出：

~~~
2014-06-22 05:29:59,115 INFO org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServicesEventType for class org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices
2014-06-22 05:29:59,409 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Adding auxiliary service httpshuffle, "mapreduce_shuffle"
2014-06-22 05:29:59,612 INFO org.apache.hadoop.mapred.ShuffleHandler: httpshuffle listening on port 13562
2014-06-22 05:31:24,846 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Got event CONTAINER_INIT for appId application_1403414881997_0003
2014-06-22 05:31:24,848 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Got event APPLICATION_INIT for appId application_1403414881997_0003
2014-06-22 05:31:24,848 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Got APPLICATION_INIT for service 
2014-06-22 05:33:04,413 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Got event CONTAINER_STOP for appId application_1403414881997_0003
2014-06-22 05:37:27,017 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.AuxServices: Got event APPLICATION_STOP for appId application_1403414881997_0002
~~~

**AuxliaryService**会在收到Application/Container初始化以及停止事件时，会进行相应的处理。

~~~
public abstract class AuxiliaryService extends AbstractService {
  /**
   * A new application is started on this NodeManager. This is a signal to
   * this {@link AuxiliaryService} about the application initialization.
   * 
   * @param initAppContext context for the application's initialization
   */
  public abstract void initializeApplication(
      ApplicationInitializationContext initAppContext);

  /**
   * An application is finishing on this NodeManager. This is a signal to this
   * {@link AuxiliaryService} about the same.
   * 
   * @param stopAppContext context for the application termination
   */
  public abstract void stopApplication(
      ApplicationTerminationContext stopAppContext);

  /**
   * Retrieve meta-data for this {@link AuxiliaryService}. Applications using
   * this {@link AuxiliaryService} SHOULD know the format of the meta-data -
   * ideally each service should provide a method to parse out the information
   * to the applications. One example of meta-data is contact information so
   * that applications can access the service remotely. This will only be called
   * after the service's {@link #start()} method has finished. the result may be
   * cached.
   * 
   * * The information is passed along to applications via
   * {@link StartContainersResponse#getAllServicesMetaData()} that is returned by
   * {@link ContainerManagementProtocol#startContainers(StartContainersRequest)}
   * 

   * 
   * @return meta-data for this service that should be made available to
   *         applications.
   */
  public abstract ByteBuffer getMetaData();

  /**
   * A new container is started on this NodeManager. This is a signal to
   * this {@link AuxiliaryService} about the container initialization.
   * This method is called when the NodeManager receives the container launch
   * command from the ApplicationMaster and before the container process is 
   * launched.
   *
   * @param initContainerContext context for the container's initialization
   */
  public void initializeContainer(ContainerInitializationContext
      initContainerContext) {
  }

  /**
   * A container is finishing on this NodeManager. This is a signal to this
   * {@link AuxiliaryService} about the same.
   *
   * @param stopContainerContext context for the container termination
   */
  public void stopContainer(ContainerTerminationContext stopContainerContext) {
  }
}
~~~

Hadoop2中提供了一个内置的AuxiliaryService(实际上，这也是我在源码中看到的唯一一个)，叫做**ShuffleHandler**，用于将Mapper的输出传输给Reducer.

NodeManager中，可以有多个**AuxiliaryServices**。有一个叫做**AuxServices**，专门用于处理这些**AuxiliaryServices**。

~~~
public class AuxServices extends AbstractService
    implements ServiceStateChangeListener, EventHandler {
  protected final Map serviceMap;
  protected final Map serviceMetaData;

  protected final synchronized void addService(String name,
      AuxiliaryService service) {
    LOG.info("Adding auxiliary service " +
        service.getName() + ", \"" + name + "\"");
    serviceMap.put(name, service);
  }
}
~~~

当**AuxServices**启动时，它会从**YarnConfiguration.NM_AUX_SERVICES**中加载**AuxiliaryService**的信息。比如，**yarn.nodemanager.aux-services**的对应的service名称是**aux-services.%s.class**。

~~~
public class AuxServices extends AbstractService
  public void serviceInit(Configuration conf) throws Exception {
    Collection auxNames = conf.getStringCollection(
        YarnConfiguration.NM_AUX_SERVICES);
    for (final String sName : auxNames) {
      try {
        Preconditions
            .checkArgument(
                validateAuxServiceName(sName),
                "The ServiceName: " + sName + " set in " +
                YarnConfiguration.NM_AUX_SERVICES +" is invalid." +
                "The valid service name should only contain a-zA-Z0-9_ " +
                "and can not start with numbers");
        Class sClass = conf.getClass(
              String.format(YarnConfiguration.NM_AUX_SERVICE_FMT, sName), null,
              AuxiliaryService.class);
        if (null == sClass) {
          throw new RuntimeException("No class defined for " + sName);
        }
        AuxiliaryService s = ReflectionUtils.newInstance(sClass, conf);
         if(!sName.equals(s.getName())) {
          LOG.warn("The Auxilurary Service named '"+sName+"' in the "
                  +"configuration is for class "+sClass+" which has "
                  +"a name of '"+s.getName()+"'. Because these are "
                  +"not the same tools trying to send ServiceData and read "
                  +"Service Meta Data may have issues unless the refer to "
                  +"the name in the config.");
        }
        addService(sName, s);
        s.init(conf);
      } catch (RuntimeException e) {
        LOG.fatal("Failed to initialize " + sName, e);
        throw e;
      }
    }
    super.serviceInit(conf);
  }
}
~~~

**AuxServices**是实现了**ServiceStateChangeListener**一个**EventHandler**接口，能够处理**AuxServicesEventType**事件。

~~~
public enum AuxServicesEventType {
  APPLICATION_INIT,
  APPLICATION_STOP,
  CONTAINER_INIT,
  CONTAINER_STOP
}
~~~

**AuxServicesEvent**中有下面的字段:

~~~
public class AuxServicesEvent extends AbstractEvent {
  private final String user;
  private final String serviceId;
  private final ByteBuffer serviceData;
  private final ApplicationId appId;
  private final Container container;
}

public abstract class AbstractEvent> 
    implements Event {
  private final TYPE type;
  private final long timestamp;
}
~~~

**AuxiliaryService**中的**handle()**方法处理事件:

~~~
public class AuxServices extends AbstractService
    implements ServiceStateChangeListener, EventHandler {
  public void handle(AuxServicesEvent event) {
    LOG.info("Got event " + event.getType() + " for appId "
        + event.getApplicationID());
    switch (event.getType()) {
      case APPLICATION_INIT:
        LOG.info("Got APPLICATION_INIT for service " + event.getServiceID());
        AuxiliaryService service = null;
        try {
          service = serviceMap.get(event.getServiceID());
          service
              .initializeApplication(new ApplicationInitializationContext(event
                  .getUser(), event.getApplicationID(), event.getServiceData()));
        } catch (Throwable th) {
          logWarningWhenAuxServiceThrowExceptions(service,
              AuxServicesEventType.APPLICATION_INIT, th);
        }
        break;
      case APPLICATION_STOP:
        for (AuxiliaryService serv : serviceMap.values()) {
          try {
            serv.stopApplication(new ApplicationTerminationContext(event
                .getApplicationID()));
          } catch (Throwable th) {
            logWarningWhenAuxServiceThrowExceptions(serv,
                AuxServicesEventType.APPLICATION_STOP, th);
          }
        }
        break;
      case CONTAINER_INIT:
        for (AuxiliaryService serv : serviceMap.values()) {
          try {
            serv.initializeContainer(new ContainerInitializationContext(
                event.getUser(), event.getContainer().getContainerId(),
                event.getContainer().getResource()));
          } catch (Throwable th) {
            logWarningWhenAuxServiceThrowExceptions(serv,
                AuxServicesEventType.CONTAINER_INIT, th);
          }
        }
        break;
      case CONTAINER_STOP:
        for (AuxiliaryService serv : serviceMap.values()) {
          try {
            serv.stopContainer(new ContainerTerminationContext(
                event.getUser(), event.getContainer().getContainerId(),
                event.getContainer().getResource()));
          } catch (Throwable th) {
            logWarningWhenAuxServiceThrowExceptions(serv,
                AuxServicesEventType.CONTAINER_STOP, th);
          }
        }
        break;
      default:
        throw new RuntimeException("Unknown type: " + event.getType());
    }
  }
}
~~~

那么，事件是如何被发送到**AuxServices**的呢？

这就要说到**ContainerManagerImpl**了:
~~~
public class NodeManager extends CompositeService 
    implements EventHandler {
  private AsyncDispatcher dispatcher;
  private ContainerManagerImpl containerManager;

  protected void serviceInit(Configuration conf) throws Exception {
    // NodeManager level dispatcher
    this.dispatcher = new AsyncDispatcher();
    containerManager =
        createContainerManager(context, exec, del, nodeStatusUpdater,
        this.aclsManager, dirsHandler);
    addService(containerManager);
    ((NMContext) context).setContainerManager(containerManager);
    dispatcher.register(ContainerManagerEventType.class, containerManager);
    dispatcher.register(NodeManagerEventType.class, this);
    addService(dispatcher);
  }
}
~~~

**ContainerManagerImpl**有它自己的**AsyncDispatcher**，它会把全部的**AuxServicesEventType**事件转发到**AuxServices**：
~~~
public class ContainerManagerImpl extends CompositeService implements
    ServiceStateChangeListener, ContainerManagementProtocol,
    EventHandler {
  private final AuxServices auxiliaryServices;
  protected final AsyncDispatcher dispatcher;
  public ContainerManagerImpl(Context context, ContainerExecutor exec,
      DeletionService deletionContext, NodeStatusUpdater nodeStatusUpdater,
      NodeManagerMetrics metrics, ApplicationACLsManager aclsManager,
      LocalDirsHandlerService dirsHandler) {
    super(ContainerManagerImpl.class.getName());
    ...
    // ContainerManager level dispatcher.
    dispatcher = new AsyncDispatcher();

    // Start configurable services
    auxiliaryServices = new AuxServices();
    auxiliaryServices.registerServiceListener(this);
    addService(auxiliaryServices);
    dispatcher.register(AuxServicesEventType.class, auxiliaryServices);
    addService(dispatcher);
  }    
~~~

**ApplicationImpl**会创建**AuxServicesEventType.APPLICATION_STOP**事件：
~~~
public class ApplicationImpl implements Application {
  static class AppFinishTransition implements
    MultipleArcTransition {

    @Override
    public ApplicationState transition(ApplicationImpl app,
        ApplicationEvent event) {

      ApplicationContainerFinishedEvent containerFinishEvent =
          (ApplicationContainerFinishedEvent) event;
      LOG.info("Removing " + containerFinishEvent.getContainerID()
          + " from application " + app.toString());
      app.containers.remove(containerFinishEvent.getContainerID());

      if (app.containers.isEmpty()) {
        // All containers are cleanedup.
        app.handleAppFinishWithContainersCleanedup();
        return ApplicationState.APPLICATION_RESOURCES_CLEANINGUP;
      }

      return ApplicationState.FINISHING_CONTAINERS_WAIT;
    }
  }

  void handleAppFinishWithContainersCleanedup() {
    // Delete Application level resources
    this.dispatcher.getEventHandler().handle(
        new ApplicationLocalizationEvent(
            LocalizationEventType.DESTROY_APPLICATION_RESOURCES, this));

    // tell any auxiliary services that the app is done 
    this.dispatcher.getEventHandler().handle(
        new AuxServicesEvent(AuxServicesEventType.APPLICATION_STOP, appId));
  }
}
~~~

其他的三个**AuxServicesEventType**，*APPLICATION_INIT*,*CONTAINER_INIT*以及*CONTAINER_STOP*，在ContainerImpl的不同阶段会抛出:
~~~
public class ContainerImpl implements Container {
  /**
   * State transition when a NEW container receives the INIT_CONTAINER
   * message.
   */
  static class RequestResourcesTransition implements
      MultipleArcTransition {
    @Override
    public ContainerState transition(ContainerImpl container,
        ContainerEvent event) {
      final ContainerLaunchContext ctxt = container.launchContext;
      container.metrics.initingContainer();

      container.dispatcher.getEventHandler().handle(new AuxServicesEvent
          (AuxServicesEventType.CONTAINER_INIT, container));

      // Inform the AuxServices about the opaque serviceData
      Map csd = ctxt.getServiceData();
      if (csd != null) {
        // This can happen more than once per Application as each container may
        // have distinct service data
        for (Map.Entry service : csd.entrySet()) {
          container.dispatcher.getEventHandler().handle(
              new AuxServicesEvent(AuxServicesEventType.APPLICATION_INIT,
                  container.user, container.containerId
                      .getApplicationAttemptId().getApplicationId(),
                  service.getKey().toString(), service.getValue()));
        }
      }
      ...
    }
  }
  /**
   * Handle the following transitions:
   * - NEW -> DONE upon KILL_CONTAINER
   * - {LOCALIZATION_FAILED, EXITED_WITH_SUCCESS, EXITED_WITH_FAILURE,
   *    KILLING, CONTAINER_CLEANEDUP_AFTER_KILL}
   *   -> DONE upon CONTAINER_RESOURCES_CLEANEDUP
   */
  static class ContainerDoneTransition implements
      SingleArcTransition {
    @Override
    @SuppressWarnings("unchecked")
    public void transition(ContainerImpl container, ContainerEvent event) {
      container.finished();
      //if the current state is NEW it means the CONTAINER_INIT was never 
      // sent for the event, thus no need to send the CONTAINER_STOP
      if (container.getCurrentState() 
          != org.apache.hadoop.yarn.api.records.ContainerState.NEW) {
        container.dispatcher.getEventHandler().handle(new AuxServicesEvent
            (AuxServicesEventType.CONTAINER_STOP, container));
      }
    }
  }
}
~~~
