---
layout: post
title: 《Apache-Hadoop-YARN--Moving-MapReduce-and-Batch-Processing-with-Apache
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 读书笔记
tags:
- 读书笔记
---
## Different Components

#### NodeManager

The NodeManager is YARN‘s per-node "worker" agent, taking care of the individual compute nodes in a Hadoop cluster. Its duties include keeping up-to-date with the ResourceManager, overseeing application containers' life-cycle management, monitoring resource usage of individual containers, tracking node health, log management, and auxiliary services that may be exploited by different YARN applications.

On startup, the NodeManager registers with the ResourceManager; it then sends heartbeats with its status and waits for instructions. Its primary goal is to manager application containers assigned to it by the ResourceManager.

YARN containers are described by a ContainerLaunchContext. This record includes a map of environemnt variables, dependencies stored in remotely accessible storage, security tokens, payloads for NodeManager services, and the command necessary to create the process. After validating the authenticity of the container lease, the NodeManager configures the environment for the container, including initializing its monitoring subsystem with the resource constraints' specified application. The NodeManager also kills containers as directed by the ResourceManager.

#### ApplicationMaster

The ApplicationMaster is the process that coordinates an application's execution in the cluster. Each application has its own unique ApplicationMaster, which is tasked with negotiating resources from the ResourceManager and working with the NodeManager to execute and monitor the tasks. In the YARN design, MapReduce is just one application framework; this design permits building and deploying distributed applications using other frameworks. For example, YARN ships with a Distributed-Shell application that allows a shell script to be run on multiple nodes on the YARN cluster.

One the ApplicationMaster is started(as a container), it will periodically send heartbeats to the ResourceManager to affirm its health and to update the record of its resource demands. After building a model of its requirements, the ApplicationMaster encodes its perferences and constraints in a heartbeat message to the ResourceManager. In response to subsequent heartheats, the ApplicationMaster will receive a lease on containers bound to an allocation of resources at a particular node in the cluster. Depending on the containers it receives from the ResourceManager, the ApplicationMaster may update its execution plan to accommodate the excess or lack of resouruces. Container allocation/deallocation can take place in a dynamic fashion as the application progress.

#### Client ResourceRequest

The client must first notify the ResourceManager that it wants to submit an application. The ResourceManager responds with an ApplicationID and information about the capabilities of the cluster that will aid the client in requesting resources.

#### ApplicationMaster Container Allocation

When the ResourceManager receives the application submission context from a client, it schedules an available container for the ApplicationMaster. This container is often called "Container0" because it is the ApplicationMaster, which must request additinal containers. If there are no applicable containers, the request must wait. If a suitable container can be found, then the ResourceManager contacts the appropriate NodeManager and starts the ApplicationMaster. As part of this step, the ApplicationMaster RPC port and tracking URL for monitoring the application's status will be established.

In response to the registration request, the ResourceManager will send information about the minimum and maximum capabilities of the cluster. At this point the ApplicationMaster must decide how to use the capabilites that are current available. Unlike some resource schedulers in which clients request hard limit, YARN allows applications to adapt to the current cluster environment.

Based on the available capabilities reported from the ResourceManager, the ApplicationMaster will request a number of containers. This request can be very specific, including containers with multiples of the resource minimum values. The ResourceManager will respond, as best as possible based on scheduling policies, to this request with container resources that are assigned to the ApplicationMaster.

As a job progresses, heartbeat and progress information is sent from the ApplicationMaster to the ResourceManager, within these heartbeats, it is possible for the ApplicationMaster to request and release containers. When the job finished, the ApplicationMaster sneds a finish message to the ResourceManager and exits.

#### ApplicationMaster-Container Manager Communication

The ApplicationMaster will indepently contact its assigned node managers and provide them with a ContainerLaunchContext that includes environment variables, dependencies located in remote storage, security tokens, and commands needed to start the actual process. When the container starts, all data files, executable, and necessary dependencies are copied to local storage on the node. Dependencies can potentially be shared between containers running the application.

Once all containers have started, their status can be checked by the ApplicationMaster. The ResourceManager is absent from the application progress and is free to schedule and monitor other resources. The ResourceManager can direct the NodeManager to kill containers. Expected kill events can happen when the ApplicationMaster informs the ResourceManager of its completion, or the ResourceManager needs nodes for another applications, or the container has exceeded its limits. When a container is killed, the NodeManager cleans up the local working directory when a job is finished, the ApplicationMaster informs the ResourceManager that the job completed successfully. The ResourceManager then informs the NodeManagerrrr to aggregate logs and cleanup container-specific files. The NodeManagers are also instructed to kill any remaining containers(including the ApplicationMaster) if they have not already exited.

#### LocalResource Visibilities

- PUBLIC: All the LocalResources that are marked PUBLIC are accessible for containers of any user.
- PRIVATE: LocalResources that are marked PRIVATE are shared among all applicatons of the same user on the node.
- APPLICATION: All the resources that are marked as having the APPLICATION scope are shared only among containers of the same application on the node.

The ApplicationMaster specifies the visibility of a LocalResource to a NodeManager while starting the container; The NodeManager itself doesn't make any decisions or classify resources. Similarly, for the container running the ApplicationMaster itself, the client has to specify visibilities for all the resources that the ApplicationMaster needs.

In case of a MapReduce application, the MapReduce Job Client decides the resource type which the corresponding ApplicationMaster then forwards to the NodeManager.

#### Lifetime of LocalResources

Different types of LocalResources have different life cycles:
- PUBLIC: LocalResources are not deleted once the container or application finished, but rather are deleted only when there is pressure on each local directory for disk capacity. The threshold for local files is dictated by the configuration property `yarn.nodemanager.localizer.cache.target-size-mb`
- PRIVATE: LocalResources follow the same file cycles as PUBLIC
- APPLICATION: Deleted immediately after the application finishes.

## Hadoop YARN architecture

#### ResourceManager components

###### Client interaction with the ResourceManager

- ClientService
  This service implements ApplicatoinClientProtocol, the basic client interface to the ResourceManager. This component handles all the remote procedure call(RPC) communications to the ResourceManager from the clients, including operations such as the following:
  - Application submission
  - Application termination
  - Exposing information about applications, queues, cluster statistics, user ACLs, and more
- Administration Service
  While ClientService is responsible for typical user invocation like application submission and termination, there is a list of activities that administrators of a YARN cluster have to perform from time to time. To make sure that administration requests don't get starred by the regular user's requests and to give the operators' commands a higher priority, all of the administrative operations are served via a seperate interface called AdministrationService.
  ResourceManager AdministrationProtocol is the communication protocol that is implemented by this component. Some of the important administrative operations are:
  - Refreshing queues
  - Refreshing the list of nodes handled by the ResourceManager
  - Adding new user-to-group mappings, adding/updating administrator ACLs, modifying the list of supers, and so on.
- ApplicationAclsManager
  The ResourceManager needs to gate the user-facing APIs like the client and administrative requets so  that they are accessible only to authorized users. This component maintains the ACLs per application and enforces them.
  An ACL is a list of users, and groups who can perform a specific operation. Users can specify the ACLs for their submitted application as part of the ApplicationSubmissionContext. These ACLs are tracked per application by the ACLsManager and used for access control whenever a request comes in. Note that irrespective of the ACLs, all administrator can perform any operation.
  The same ACLs are fransferred over to the ApplicationMaster so that the ApplicationMaster itself can use them for users accessing various services running inside the ApplicationMaster. The NodeManager also receives the same ACLs as part of ContainerLaunchCOntext when a container is launched which then uses them for access control to serve requests about the applications/containers, mainly about their status, application logs, etc.

- ResourceManager Web Application and Web Services
  The ResourceManager has a web application that exposes information about the state of the cluster, metrics, list of active, healthy, and unhealthy nodes, list of applications, their state and status; 

###### Application interaction with the ResourceManager

The following describes how the ApplicationMasters interact with the ResourceManager once they have started.

- ApplicationMastersService
  This component responds to requets from all the ApplicationMasters. It implements ApplicationMasterProtocol, which is the one and only protocol that ApplicationMasters use to communication with the ResourceManager. It is responsible for the following fasks:
    - Registration of new ApplicationMasters
    - Termination/unregistering of requets from any finishing ApplicationMaster
    - Authorizing all requets from various ApplicationMasters to make sure that only valid ApplicationMasters are sending requets to the corresponding Application entry residing in the ResourceManager
    - Obtaining container allocation and deallocation requets from all running ApplicationMasters and forwarding them asynchrously to the YarnScheduler.
  The ApplicationMasterService has additional logic to make sure that, at any point in time, only one thread in any ApplicationMaster can send requets to the ResourceManager. All the RPCs from ApplicationMasters are serialized on the ResourceManager, so it is expected that only one thread in the ApplicationMaster will make these requests.

- ApplicaitonMaster Liveliness Monitor
  To help manage the list of live ApplicationMasters and dead/non-responding ApplicationMasters, this monitor keeps track of each ApplicaitonMaster and its last heartbeat time. Any ApplicationMaster that does not produce a heartbeat within a configured internal of time- by default, 10 minutes - is demed dead and is expired by the ResourceManager. All containers currently running/allocated to an expired ApplicationMaster are marked as dead. The ResourceManager reschedules the same application to run a new ApplicationAttempt on a new container, allowing up to a maximum of two such attempts by default.

###### Interaction of Nodes with the ResourceManager

  The following components in the ResourceManager interact with the NodeManagers running on cluster nodes.

- Resource Tracker Service
  NodeManagers periodically send heartbeat to the ResourceManager, and this component of the ResourceManager is responsible for responding to such RPCs from all the nodes. It implements the ResourceTracker interface to which all NodeManagers communication. Specifically, it is responsible for the following tasks:
    - Registering new nodes
    - Accepting node heartbeats from previously registered nodes
    - Ensuring that only valid nodes can interact with the ResourceManager and rejecting any other nodes.
  Following a successful registration, in its registration response the ResourceManager will send security-related master keys needed by NodeManagers to authenticate container-related requets from the ApplicationMasters.
  NodeManagers need to be able to validate NodeManager tokens and container tokens that are submitted by ApplicationMasters as part of container-launch requests. The underlying master keys are rolled over every so often for security purposes; thus, no further heartbeats, NodeManagers will be notified of such updates whenever they happen.
  The ResponseTrackerService forwards a valid node-heartbeat to the YARNScheduler, which then makes scheduling decissions based on freely available resources on that node and resource requirements from various applications.

- NodeManagers Liveliness Monitor
  To keep track of live nodes and specifically identify any dead nodes, this component keeps track of each node's identifier and its last heartbeat time. Any node that doesn't send a heartbeat within a configured interval of time - by default, 10 minutes - is deemed dead and is expired by the ResourceManager. All the containers currently running on an expired node are marked as dead, and no new containers are scheduled in such node. Once such a node restarts the registers, it will again be considered for scheduing.

- NodesListManager
  The nodes-list manager is a collection in the ResourceManager's memory of both valid and excluded nodes. It is responsible for reading config files specified via the 'yarn.resourcemanager.nodes.include-path' and 'yarn-resourcemanager.nodes.execlude-path' configuration properties and seeding the initial list of nodes based on those files. It also keeps track of nodes taht are explicitly decommissioned by administrators as time processes.

###### Core ResourceManager Components

- ApplicationsManager
The ApplicationsMaster is responsible for maintaining a collection of submitted applications. After application submission, it first validates the application's specifications and rejects any application that requests unsatisfiable resources for its ApplicaitonMaster. It then ensures that no other application was already submitted with the same application ID - a scenario that can be caused by an erroneous or a malicious client. Finally, it forwards the admitted application to the scheduler.
This component is also responsible for recording and managing finished applications for a while before they are completely evacuated  from the ResourceManager's memory. When an application finishes, it places an ApplicationSummary in the daemon's log file. The ApplicationSummary is a compact representation of application information at the time of completion.
Finally, the ApplicationsManager keeps a cache of completed application long after applications finish to support user's requests for application data. The configuration property 'yarn.resourcemanager.max-completed-applications' controls the maximum number of such finished applications that the ResourceManager remembers, at any point of time. The cache is a first-in, first out list, with the oldest applications being moved out to accommodate freshly finished applications.

- ApplicationMasterLauncher
In YARN, while every other container's launch is initiated by an ApplicationMaster, the ApplicationMaster itself is allocated and prepared for launch on a NodeManager by the ResourceManager itself. The ApplicationMaster launcher is responsible for this job. This component maintains a thread pool to set up the environment and to communicate with NodeManagers so as to launch ApplicationMasters of newly submitted applications as well as applications for which provious ApplicationMaster attempts failed for some reason. It is also responsible for talking to NodeManager about cleaning up the ApplicationMaster - mainly killing the process by signaling the corresponding NodeManager when an application finishes normally or is forcefully terminated.

- YarnScheduler
The YarnScheduler is responsible for allocating resources to the various running applications subject to constraints of capacities, queues, and so on. It performs its scheduling function based on the resource requirements of the applications, such as memory, CPU, disk, and network needs.

- ContainerAllocationExpirer
This component is in charge of ensuring that all allocated containers are eventually used by ApplicationMasters and subsequently launched on the corresponding NodeManagers. ApplicationMasters run as untrusted user code and may potentially hold on to allocations without using them; as such, they can lead to under-utilication and abuse of a cluster's resources. To address this, the ContainerAllocationExpirer maintains a list of containers that are allocated but still not used on the corresponding NodeManagers. For any container, if the corresponding NodeManager doesn't report to the ResourceManager that the container has started running within a configured internal of time, the container is deemed dead and is expired by the ResourceManager.
In addition, independently NodeManagers look at this expiry time, which is encoded in the ContainerToken tried to a container, and reject containers that are submitted for launching after the expiry time elapses. Obviously, this feature depends on the system clocks being synchronized across the ResourceManager and all NodeManagers in the system.

###### Security-related components in the ResourceManager

The ResourceManager has a collection of components called SecretManagers that are charged with managing the tokens and secret keys that are used to authenticate/authorize requests on various RPC interfaces.

- ContainerTokenSecretManager
This SecretManager is responsible for managing ContainerTokens - a special set of tokens issued by the ResourceManager to an ApplicationMaster so that if can use an allocated container on a specific node. This ResourceManager specific component keeps track of the underlying secret keys and rolls the keys over every so often.
ContainerTokens are a security tool used by the ResourceManager to send vital information related to starting a containerto NodeManagers through the ApplicationMaster. This information cannot be sent directly to a NodeManager without causing significant latencies. The ResourceManager can construct ContainerTokens only after a container is allocated, and the information to be encoded in a ContainerToken is available only after this allocation. Waiting for NodeManager to acknowledge the token before ApplicationMasters can get the allocated container in a nonstarted. For this reason, they are routed to the NodeManagers through the ApplicationMasters.
ResourceManager encrypts vital container-related information into a container token before sending it to the ApplicationMaster.
A container token consists of the following fields:
  - Container ID
  - NodeManager address
  - Application Submitter
  - Resource
  - Expiry timestamp
    - NodeManagers look at this timestamp to determine if the container token passed is still valid. Any containers that are not used by the ApplicationMasters until after this expiry time is reached will be automatically cancelled by YARN.
    - For this feature to work, the clocks on the nodes running the ResourceManager and the NodeManagers must be in sync.
    - When the ResourceManager allocates a container, it also determines and sets its expiry time based on a cluster configuration, defaulting to 10 minutes.
    - When administrators set the expiry internal configuration, it should not be set to a very low value, beacuse ApplicationMasters may not have enough time to start containers before they are expired, or to a very high value, because doing so permits rogue ApplicationMasters to allocate containers but not use them, which hurts cluster utilization.
    - If a container is not used before it expires, then the NodeManager will simply reject any start-container requests using this token. The NodeManager also has a cache of recently started containers to prevent ApplicationMasters from using the same token in a rapid manner on very short-lived containers.
  - Master key identifier
    - THe ResourceManager generates a secret key and assigns a key ID to uniquely identify this key. This secret key, along with its ID, is shared with every NodeManager, first as a part of each node's registration and then during subsequent heartbeats whenever the ResourceManager rolls over the keys for securith reasons. The key rollover period is a ResourceManager configurable value, but defaults to a day.
    - Whenever the ResourceManager rolls over the underlying keys, they aren't immediately used to generate new tokens; thus there is enough time for all the NodeManagers in the cluster to learn about the rollover. As NodeManagers emit heartbeats and learn about the new key, or once the activation period expires, the ResourceManager replaces its older key with a newly created key. Thereafter, it used the new key only for generating container tokens. This action vation period is set to be 1-5 times the node-expiry interval.
    - As you can see, there will be times before key activation when NodeManagers may receive tokens generated using different keys. In such a case, even when the ResourceManager instructs NodeManager that a key has rooled over, NodeManager continue to remember both the current key and the previous key, and use the correct key based on the master key ID present in the token.
  - ResourceManager identifier
    It is possible that the ResourceManager might restart after allocating a container but before the ApplicationMaster can reach the NodeManager to start the container. To ensure both the new ResourceManager and the NodeManagers are able to recognize containers from the old instance of ResourceManager separately from the ones allocated by the new instance, the ResourceManager identifier is encoded into the container token. At the time of this writing, the ResourceManager on restart will kill all of the previously running containers; in a similar rein, NodeManager simply reject containers issued by the order ResourceManager.

- AMRMToken SecretManager
To avoid the possibility of arbitrary processes maliciously initating a real ApplicationMaster uses per-ApplicationAttempt tokens called AMRMTokens. This secret manager saves each token locally in memory until an ApplicationMaster finishes and uses it to authenticate any request coming from a valid ApplicationMaster process.
ApplicationMasters can obtain this token by loading a credentials file localized by YARN. The location of this file is determined by the public constant `ApplicationContants.CONTAINER_TOKEN_FILE_ENV_NAME`.
Unlike the container tokens, the underlying master key for AMRMTokens doesn't need to be shared with any other entity in the system. Like the container tokens, the keys are rolled every so often for security reasons, but there are no corresponding activation periods.

- NMTokenSecretManager
ApplicationMasters use NMTokens to manage one connection per NodeManager and use it to send all requets to that node.
  - The ResourceManager generates one NMToken per application attempt per NodeManager
  - Whenever a new container is created, ResourceManager issues the ApplicationMaster an NMToken corresponding to that node. ApplicationMasters will get NMTokens only for those NodeManagers on which they started containers.
  - As a network optimization, NMTokens are not sent to the ApplicationMasters foreach and every allocated container, but only for the first time or if NMTokens have to be invalidated due to the rollover of the underlying master key.
  - Whenever an ApplicationMaster receives a new NMToken, it should replace the existing token, if present, for that NodeManager with the newer token. A library, NMTokenCache, is available for the token management.
  - ApplicationMasters are always expected to use the latest NMToken, and each NodeManager accepts only one NMToken from any ApplicationMaster. If a new NMToken is received from the ResourceManager, then older connections for correspdong NodeManagers should be closed and a new connection should be created with the latest NMToken. If connections created with older NMTokens are then used for launching newly assigned containers, the NodeManagers simply reject them.
  - As with container tokens, NMTokens issued for one ApplicationMaster cannot be used by another. To make this happen, the application attemptID is encoded into the NMTokens.

- RMDelegationToken SecretManager
  This component is a ResourceManager specific delegation token secret manager. It is responsible for generating delegation tokens to clients, which can be passed on to processes that wish to be able to talk to the ResourceManager but are not kerberos authenticated.

- DelegationToken Renewer
  In secure mode, the ResourceManager is kerberos authenticated and so provides the service of renewing file system tokens on behalf of the applications. This component renews tokens of submitted applications as long as the application runs and until the tokens can no longer be renewed.

#### NodeManager

It responsibilties include the following tasks:
  - Keeping up-to-date with the ResourceManager
  - Tracking node health
  - Overseeing containers' life-cycle management; monitoring resource usage of individual containers
  - Managing the distribuetd cache
  - Managing the logs generated by containers
  - Auxiliary services that may be exploited by different YARN applications

###### Overview of the NodeManager components

After registering with the ResourceManager. the NodeManager periodically sends a heartbeat with its current status and receives instructions, if any, from the ResourceManager. When the scheduler gets to process the node's heartbeat, containers are allocated against that NodeManager and then are subsequently returned to the ApplicationMasters when the ApplicationMasters themselves send a heartbeat to the ResourceManager.

Before actually launching a container, the NodeManager copies all the necessary libraries - data files, executables, tarballs, jar fiels, shell scripts, and so on - to the local file system. The downloaded libraries may be shared between containers of a specific application via a local applicaiton-level cache, between containers launched by the same user via a local user-level cache, and even between users via a public cache, as can be specified in the ContainerLaunchContext. The NodeManager eventually garbage-collects libraries that are not in use by any running containers.

The NodeManager may also kill containers as directed by the ResourceManager. Containers may be killed in the following situations:
  - The ResourceManager sends a signal that an application has completed
  - The scheduler decides to preempt it for another application or user
  - The NodeManager detects that the container exceeded the resource limits as specified by its ContainerToken

Whenever a container exists, the NodeManager will clean up its working directory in local storage. When an application completes, all resources owned by its containers are cleaned up.

In addition to starting and stopping containers, cleaning up after exited containers, and managing local resources, the NodeManager offers other local services to containers running on the node. For example, the log aggregation service uploads all the logs written by the application's containersr to stdout and stderr to a file system once the application completes.

###### NodeManager components

![](https://upload-images.jianshu.io/upload_images/4108852-08421c4b89886ce8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- NodesStatusUpdater
On startup, this component registers with the ResourceManager, sends information about the resources available on this node, and identifies the ports at which the NodeManager's web server and the RPC server are listening. As part of the registration, the ResourceManager sends the NodeManager security-related keys needed by the NodeManager to authenticate future container requests from the ApplicationMasters. Subsequent NodeManager-ResourceManager communication provides the ResourceManager with any updates on existing containers' status, new containers started on the node by the ApplicationMasters, containers that have completed, and so on.
In addition, the ResourceManager may signal the NodeManager via this component to potentially kill currently running containers. Finally, when any application finishes on the ResourceManager, the ResourceManager signals the NodeeManager to cleanup various application-specific entities on the NodeManager, and then instiate and finish the per-application logs' aggregation onto a file system.

- ContainerManager
  1. RPCServer
    Accept requets from ApplicationMasters to start new containers, or to stop running one. All the operations performed on containers running on this node are recorded in an audit log, which can be post processed by security tools.
  2. ResourceLocalizationService
    The ResourceLocalizationService is responsible for securely downloading and organizing various file resources needed by containers. It tries its best to distribute the files across all the available disks. It also enforces accees control restrictions on the downloaded files and put appropriate usage limits on them.
    Localization of PUBLIC Resources is taken care of by a pool of threads called PublicLocalizers. PublicLocalizers run inside the address space of the NodeManager itself. The number of PublicLocalizer threads is controlled by the configuration property 'yarn.nodemanager.localizer.fetch.threadcount'，which sets the maximum parallelism during downloading of PUBLIC resources to this thread count. While localizing PUBLIC resources, the localizer validates that all the requested resources are, indeed,  PUBLIC by checking their permissions on the remote file system. Any LocalResource that doesn't match that condition is rejected for localization. Each PublicLocalizer uses credentials passed as part of ContainerLaunchContext to securely copy the resources from the remote file system.
    Localization of PRIVATE/APPLICATION Resources happen in a sparate process called ContainerLocalizer. Every ContainerLocalizer process is managed by a single thread in NodeManager called LocalizerRunner. Every container will trigger one LocalizerRunner it has any resources that are not yet downloaded.
    LocalResourceTracker is a per-user or per-application object that tracks all the LocalResources for a given user or an application.
    When a container first requests a PRIVATE/APPLICATION LocalResource, if it is not found in LocalResourcesTracker, it is added to pending resources list.
    A LocalizerRunner may be created depending on the need for downloading something new.
    The LocalizerRunner starts a LinuxContainerExecutor. The LCE is a process running as application submitter, which then executes a ContainerLocalizer. The ContainerLocalizer works as follows:
      - Once started, the ContainerLocalizer starts a heartbeat with the NodeManager process.
      - On each heartbeat, the LocalizerRunner either assigns one resource at a time to a ContainerLocalizer or asks it to die. The ContainerLocalizer informs the LocalizerRunner about the status of the download.
      - If it fails to download a resource, then that particular resource is removed from LocalResourcesTracker and the container eventually is marked as failed. When this happens, the LocalizerRunner stops the running ContainerLocalizers and exits.
      - If it is a successful download, then the ContainerRunner gives a ContainerLocalizer another resource again and again, continuing to do so until all pending resources are successfully downloaded.
    Each ContainerLocalizer doesn't support parallel downloading of multiple PRIVATE/APPLICATION resources. In addition, the maximum parallelism is the number of containers requested for the same user on the same NodeManager at that point of time. The worst case for this process occurs when an ApplicationMaster itself is starting. If the ApplicationMaster needs any resources to be localized then, they will be downloaded serially before its container starts.
  3. ContainersLauncher
    The ContainersLauncher maintains a pool of threads to prepare and launch containers as quickly as possible. It also cleans up the containers' processes when the ResourceManager sends such a request through the NodeStatusUpdater or when the ApplicationMasters send requests via the RPC server. The launch or cleanup of a container happends in one thread of the thread pool, which will return only when the corresponding operation finishes. Consequently, launch or cleanup of one container doesn't affect any other operations and all container operations are isolated inside the NodeManager process.
  4. AuxiliaryServices
    An administrator may configure the NodeManager with a set of pluggable, auxiliary services. The NodeManager provides a framework for extending its functionality by configuring these services. This feature allows per-node custom services that specific frameworks may require, yet places them in a local "sandbox" separate from the rest of the NodeManager. These services must be configured before the NodeManager starts. AuxiliaryServices are notified when an application's first container starts on the node, whenever a container starts or finishes, and finally when the application is considered to be complete.
    While a container's local storage will be cleaned up after it exists, it can promote some output so that it will be preserved until the application finishes. In this way, a container may produce data that persists beyond the life of the container, to be managed by the node. This property of output persistence, together with auxiliary services, enables a powerful features. One important use-case that tasks advantage of this feature is Hadoop MapReduce. For Hadoop MapReduce applications, the intermediate data are transferred between the map and reduce tasks using an auxiliary service called ShuffleHandler. As mentioned earsier, the CLC allows ApplicationMasters to address a payload to auxliary services. MapReduce applications use this channel to pass tokens that authenticate reduce tasks to the Shuffle Service.
    When a container starts, the service information for auxiliary services is returned to the ApplicationMaster so that the ApplicationMaster can use this information to take advantage of any available auxiliary services. As an example, the MapReduce framework gets the ShuffleHandler's port information, which it then passes on to the reduce tasks for shuffling map outputs.
  5. ContainersMonitor
    After a container is launched, this component starts observing its resource utilization while the container is running. To enforece isolation and fair sharing of resources like memory, each container is allocated some amount of such a resource by the ResourceManger. The ContainersMonitor monitors each container's usage continuously. If a container exceeds its allocation, this component signals the container to be killed. This check is done to prevent any runaway container from adversly affecting other wellbehaved containers running on the same node.
  6. Log Handler
    The LogHandler is a pluggable component that offers the option of either keeping the containers' log on the local disks or zipping them together and uploading them onto a file system.

- ContainerExecutor
  Interact with the underlying operating system to securely place files and directories needed by containers and subsequently to launch and clean up processes corresponding to containers in a secure manner.

- NodeHealthCheckService
  The NodeHealthCheckerService provides for checking the health of a node by running a configured script frequently. It also monitors the health of the disks by creating temporary files on the disks every so often. Any changes in the health of the system are sent to NodeStatusUpdater, which in turn passes the information to the ResourceManager.

- ApplicationACLsManager
  Maintains the ACL for each application and enforces the access permissions whenever such a request is received.

- ContainerTokenSecretManager
  Verifies various incoming requests to ensure that all the start container requests are properly authorized by the ResourceManager.

- NMTokenSecretManager
  Verifies all incoming API calls to ensure that the requests are properly authenticated using NMTokens.

- WebServer
  Exposes the list of applications, containers running on the node at a given point of time, node-health-related information, and the logs produced by the containers.

###### Impotant NodeManager Functions

- Container Launch
  To facilitate container launch, the NodeManager expects to receive detailed information about a container's run time, as part of the total container specification. This includes the container's command line, environment variables, a list of resources required by the container, and any security tokens.
  On receiving a container-launch request, the NodeManager first verifies this request and determines if security is enabled, so as to authorize the user, correct resources assignment, and other aspects of the request. The NodeManager then performs the following set of steps to launch the container.
    - A local copy of all the specified resources is created.
    - Isolated work directories are created for the container, and the local resources are made available in these directories by way of symbolic links to the downloaded resources.
    - The launch environment and command line are used to start the actual container.

- User Log Management and Aggregation

- MapReduce Shuffle AuxiliaryService
  The Shuffle functionality required to run a MapReduce application is implemented as an auxiliary service. This service starts up a Netty web server, and knows how to handle MapReduce - specific shuffle requests from reduce tasks. The MapReduce ApplicationMaster specifies the service ID for the shuffle service, along with security tokens that may be required. The NodeManager provides the ApplicationMaster with the port on which the shuffle service is running; this information is then passed to the reduce tasks.

#### ApplicationMaster

###### Overview

![](https://upload-images.jianshu.io/upload_images/4108852-3c4c306cfcfb1124.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

The process starts when an application submits a request to the ResourceManager. Next, the ApplicationMaster is started and registers with the ResourceManager. The ApplicationMaster then requests containers from the ResourceManager to perform actual work. The assigned containers are presented to the NodeManager for use by the ApplicationMaster. Computation takes place in the containers, which keep in contact with the ApplicationMaster as the job progresses. When the application is complete, containers are stopped and the ApplicationMaster is unregisterred from the ResourceManager.

Once successfully launched, the ApplicationMaster is responsible for the following tasks:
- Initializing the process of reporting liveliness to the ResourceManager
- Computing the resource requirements of the application
- Translating the requirements into ResourceRequests that are understood by the YARN scheduler
- Nepotiating those resource requests with the scheduler
- Using allocated containers by working with the NodeManager
- Tracking the status of running containers and monitoring their process
- Reacting to container or node failures by requesting alternative resources from the scheduler of needed

###### Liveliness

The first operation that any ApplicationMaster has to perform is to register with the ResourceManager. As part of the registration, ApplicationMasters can inform the ResourceManager about an IPC address and/or a web URL.

In the registration response, the ResourceManager returns information that the ApplicationMaster can use, such as the minimum and maximum sizes of resources that YARN accepts, and the ACLs associated with the application that are set by the user during application submission. The ApplicationMaster can use these ACLs for authorizing user requests on its own client-facing service.

Once registered, an ApplicationMaster periodically needs to send heartbeats to the ResourceManager to affirm its liveliness and health.

###### Resource Requirements

Resource requirements are referred to as static when they are decided at the time of application submission and when, once the ApplicationMaster starts running, there is no change in that specification. For example, in the case of MapReduce, the number of mappers and reducers.

When dynamic response requirements are applied, the ApplicationMaster may choose how many resources to request at run time based on criteria such as user hints, availability of cluster resources, and business logic.

###### Scheduling

  When an ApplicationMaster accumulates enough resource requests or a timer expires, it can send the requests in a heartbeat message, via the 'allocate' API, to the ResourceManager. The 'allocate' call is the single most important API between the ApplicationMaster and the scheduler. It is used by the ApplicationMaster can invoke the 'allocate' API; all such calls are seralized on the ResourceManager per ApplicationAttempt. Because of this, if multiple threads asks for resources via the 'allocate' API, each thread may get an inconsistent view of the overall resource requests.

###### Scheduling Protocol and Locality

1.  ResourceRequests
  The ResourceRequest object is used by the ApplicationMaster for resource requests. It includes the following elements:
    - Priority of the request
    - The name of the resource location on which the allocation is desired. It currently accepts a machine or a rack name. A special value of "*" signifies that any host/rack is acceptable to the application.
    - Resource capability, which is the amount or size of each container required for that request.
    - Number of containers, with respect to the specifications of priority and resource location, that are required by the application.
    - A Boolean relaxLocality flag(defaults to true), which tells the ResourceManager if the application wants locality to be loose(i.e., allow fall-through to rack or "*" in case of no local containers) or strict(i.e., specify hard constraints on container placement).
  
###### Priorities

Higher-priority requests of an application are served first by the ResourceManager before the lower-priority requests of the same application are handled. There is no cross-application implication of priorities.

###### Launching Containers

Once the ApplicationMaster obtains containers from the ResourceManager, it can then proceed to actual launch of the containers. Before launching a container, it first has to construct the ContainerLaunchContext object according to its needs, which can include allocated resource capability, security tokens, the command to be executed to start the container, and more. It can either launch containers one by one by communicating to a NodeManager, or it can batch all containers on a single node together and launch them in a single call by providing a list of StartContainerRequests to the NodeManager.

The NodeManager sends a response via StartContainerResponse that includes a list of successfully launched containers, a container ID-to-exception map for each failed StartContainerRequest in which the exception indicates errors per container, and an allServicesMetaData map from the names of auxiliary services and their corresponding metadata.

The ApplicationMaster can also get updated statuses for submitted but still to be launched containers as weel as already launched containers.

The ApplicationMaster can also request a NodeManager to stop a list of containers running on that node by sending a StopContainersRequest that includes the container IDs of the containers that should be stopped. The NodeManager sends a repsonse via StopContainersResponse, which includes a list of container IDs of successfully stopped containers as well as a container ID-to-exception map for each failed request in which the exception indicates errors from the particular container.

When an ApplicaitonMaster exits, depending on its submission context, the ResourceManager may choose to kill all the running containers that are not explicitly terminated by the ApplicationMaster itself.

###### Completed Containers

When containers finish, the ApplicationMasters are informed by the ResourceManager about the event. Because the ResourceManager does not interpret the container status, the ApplicationMaster determines the sematics of the success or failure of the container exit status reported through the ResourceManager.

Handling of container failures is the responsibility of the applications/frameworks. YARN is responsible only for providing information to the applicaitons/framewoks. The ResourceManager collects information about all the finished containers as part of the 'allocate' API's response, and it returns this information to the corresponding ApplicationMaster. It is up to the ApplicationMaster to look at information such as the container status, exit code, and diagnostics information and act on it appropriately. For example, when the MapReduce ApplicationMaster learns about container failures, it retries map or reduce tasks by requesting new containers from the ResourceManager until a configured number of attempts fail for a single task.

###### ApplicationMaster failures and recovery

The ApplicationMaster is also tasked with recovering the application after a restart that was due to the ApplicationMaster's own failure. When an ApplicationMaster fails, the ResourceManager simply restarts an application by launching a new ApplicationMaster for a new ApplicationAttempt; it is the responsibility of the new ApplicationMaster to recover the application's previous state. This goal can be achieved by having the current ApplicationAttempts persist their current state to external storage for use by future attempts. Any ApplicationMaster can obvisously just run the appliaction from scratch all over again instead of recovering the part state. The Hadoop MapReduce framework's ApplicationMaster recovered its completed tasks, but running tasks as well as the tasks that completed during ApplicationMaster recovery would be killed and rerun.

###### Coordination and output commit

If a framework supports multiple containers contending for a resource or an output commit, ApplicationMaster should provide synchronization primitives for them, so that only one of those containers can access the shared resource, or it should promote the output of one while the other is ordered to wait or abort. The MapReduce ApplicationMaster defines the number of multiple attempts per task that can potentially run concurrently; it also provides APIs for tasks so that the output-commit operation demonstrates consistency.

YARN can't guarantee there is only one application attempt because the network partitions. So Application writers must be aware of this possibility, and should code their applications and frameworks to handle the potential multiple-writer problems.

###### Information for clients

###### Security

If the application exposes a web service or an HTTP/socket/RPC interface, it is also responsible for all aspects of its secure operation. YARN newely secures its deployment.

###### Cleanup an ApplicationMaster exit

When an ApplicationMaster is done with all its work, it should explicitly unregister with the ResourceManager by sending a FinishApplicationRequest. Similar to registration, as part of this request ApplicationMasters can report IPC and web URLs where clients can go once the application finishes and the ApplicaitonMaster is no longer running.

Once an ApplicationMaster's 'finish' API causes the application to finish, the ApplicationMaster exits on its own or the ApplicationMaster liveliness interval is reached. This is done so as to enable ApplicationMasters to do some cleanup after the finish API is successfully recorded on the ResourceManager.

#### YARN Containers

###### Container Environment

- The AplicationMaster should describe all libraries and other dependencies needed by a container for its start-up as part of its ContainerLaunchContext. That way, at the time of the container launch, such dependencies will already be downloaded by the localization in the NodeManager and be ready for linking directly.

- Input/output paths and file-system URLs are a part of the configuration that is beyond the control of YARN. Applications are required to propagate this information themselves. THere are multiple ways one can do this: 
  - Environment variables
  - Command-line parameters
  - Separate configuration files that are themselves passed as local resources.

- Local directories where contianers are write some outputs are determined by the environment variable `ApplicationConstants.Environment.LOCAL_DIRS`

- Containers that need to log output or error statements to files need to make use of the log directory functionality. The NodeManager decide the location of log directories at run time. Because of this, a container's command line or its environment variables should point to the log directory by using a specialized marker defined by `ApplicationConstants.LOG_DIR_EXPANSION_VAR`. This marker will be automatically replaced with the correct log directory on the local file system when a container is launched.

- The username, home directory, container ID, and some other environment-specific information are exposed as environment variables by the NodeManager; Containers can simply look them up in their environment. All such environment variables are documented by the `ApplicationConstants.Environment` API.

- Security-related tokens are all available on the local file system in a file whose name is provided in the container's environment, keyed by the name `ApplicaitonConstants.CONTAINER_TOKEN_FILE_ENV_NAME`. Containers can simply read this file and load all the credentials into memory.

Dynamic information includes settings that can potentially change during the lifetime of a container. It is composed of things like the location of the parent ApplicaitonMaster and the location of map outputs for a reduce task. Most of this information is the responsibility of the application-specific implementation. SOme of the options includes the following:

- The ApplicationMaster's URL can be passed to the container via environment variables, a command-line argument, or the configuration, but any dynamic changes to it during fail-over can be found by directly communicating to the ResourceManager as a client.

- The ApplicationMaster can coordinate the locations of the container's output and the corresponding auxiliary services, making this information available to other containers.

- The location of the HDFS NameNode may be obtained from a dynamic plug-in that performs a configuration-based lookup of where an active NameNode is running.

###### Communication with the ApplicationMaster

When containers exit, ApplicationMaster will eventually learn about their completion status, either directly from the NodeManager or via the NodeManager-ResourceManager-ApplicationMaster channel for status of completed containers.

If an application needs its container to be in communication with its ApplicaitonMaster, however, it is entirely up to the applicaiton/framework to implement such a protocol.

###### Summary for Application-writers

- Submit the application by passing a ContainerLaunchContext for the ApplicationMaster to the ResourceManager.

- After the ResourceManager starts the ApplicationMaster, the ApplicaitonMaster should register with the ResourceManager and periodically report its liveliness and resource requirements over the wire.

- Once the ResourceManager allocates a container, the ApplicationMaster can construct a ContainerLaunchContext to launch the container on the corresponding NodeManager. It may also monitor the status of the running container and stop it when the work is done. Monitoring the progress of work done inside the container is strictly the ApplicaitonMaster's responsibility.

- Once the ApplicationMaster is done with its overall work, it should unregister from the ResourceManager and exit cleanly.

- Optionally, frameworks may add control flow between their containers and the ApplicationMaster as well as between their own clients and the ApplicationMaster to report status information.

## CapacityScheduler

#### Ideas

1. Elesticity with multitenancy
2. Security: ACLs
3. Resource Awareness
4. Granular Scheduling
5. Locality
6. Scheduing Policies

#### Queues

Every queue in the CapacityScheduler has the following properties:
- A short queue name
- A full queue path name
- A list of child queues and applications associated with them
- Guaranteed capacity of the queue
- Maximum capacity to which a queue can grow
- A list of active users and corresponding limits of sharing between users
- State of the queue
- ACLs governing the access to the queue

###### Hierarchical Queues

1. Key Characteristics
Queues are of two types: parent queues and leaf queues.
Parent queues enable the management of resources across organizations and suborganizatins. They can container more parent queues or leaf queues. They do not themselves accept any application submissions directly.
Leaf queues denote the queues that live under a parent queue and accept applications. Leaf queues do not have any more child queues.

2. Scheduling Among Queues
The scheduling olgorithm works as follows:
- At every level in the hierarchy, every parent queue keeps the list of its child queues in a sorted manner based on demand. The sorting of the queues is determainted by the currently used fraction of each queue's capacity(or the queue names if the utilization of any two queues is equal) at any point in time.
- The ROOT queue understands how the cluster capacity has to be distributed among the first level of parent queues and invokes scheduling an each of its child queues.
- Every parent queue also tries to follow the same capacity constraints for all of its child queues and schedules them accordingly
- Leaf queues hold the list of active applications, potentially from multiple users, and schedule resources in a FIFO manner while simultaneously respecting the limits on how much a single user can take within that queue.

Example hierarchies for use by CapacityScheduler:
![](https://upload-images.jianshu.io/upload_images/4108852-4735365b63a640fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

There are limitations on how one can name the queues. To avoid confusion, the CapacityScheduler doesn't allow two leaf queues to have the same name across the whole hierarchy.

###### Queue Access Control

Although applicaiton submission can really happend only at the leaf queue level, an ACL on a parent queue can be set to control admittance to all the descendant queues.

Access Control Lists can be configured by following format: a comma-separated list of users, followed by a space separator, followed by a comma-separated list of groups, for example:
  `user1, user2 group1, goup2`

If it is set to "*", all users and groups are allowed to perform the operation guarded by the ACL in question. If it is set to **" "**, no users or groups are allowed to perform the operation.

###### Capacity Management with Queues

Assume the administrators decide to share the cluster resources between the grunpy-engineers, finance-wizards, and marketing-moguls in a **6:1:3** ratio, the corresponding queue configuration will be as follows:
~~~
{
  "yarn.scheduler.capacity.root.grumpy-engineers.capacity": "60",
  "yarn.scheduler.capacity.root.finance-wizards.capacity": "10",
  "yarn.scheduler.capacity.root.marketing-moguls.capacity": "30"
}
~~~

During scheduling, queues at any level in the hierarchy are sorted in the order of their current used capacity and available resources are distribtued among them, starting with those queues that are the most under-served at that point in time. With respect to just capacities, the resource scheduling has the following flow:
1. The more under-served the queues, the higher the priority that is given to them during resource allocation. The most under-served queue is the queue with the smallest ratio of used capacity to the total cluster capacity.
  1.1 The used capacity of any parent queue is defined as the aggregate sum of used capacity of all the descendant queues recursively.
  1.2 The used capacity of a leaf queue is the amount of resources that is used by allocated containers of all applications running in that queue.

2. Once it is decided to give a parent queue the freely available resources, further similar scheduling is done to decide recursively as to which child queue gets to use the resources based on the same concept of used capacities.

3. Scheduling inside a leaf queue further happens to allocate resources to applications arriving in a FIFO order.
  3.1 Such scheduling also depends on locality, user level limits, and application limits
  3.2 Once an application within a leaf queue is chosen, scheduling happens within an application, too. Applications may have different resource requests at different priorities.

4. To ensure elasticity, capacity that is configured but not utilized by any queue due to lack of demand is automatically assigned to the queues that are in need of resources.

To prevent a queue increasing too much and hold on the whole resources, and make other application block, administrators can set the following limit:

~~~
{
  "yarn.scheduler.capacity.root.grumpy-engineers.infinite-monkeys.maximum-capacity": "40"
}
~~~

Capacities and maximum capacities can be dynamically changed at run time using the rmadmin "refresh queues" functionality.

###### User Limits

Leaf queues have the additional responsibiltiy of ensuring fairness with regard to scheduling applications submitted by various users in that queue. The capacity scheduler places various limits on users to enforce this fairness. Recall that applications can only be submitted to leaf queues in the capacity scheduler; thus, parent queue do not have any role in enforcing user limits.

Assume the queue capacity needs to be shared among not more than five users in the thrifty-treasurers queue. When you account for fairness, this results in each of those five users being given an equal share(20%) of the capacity of the "root.finance-wizards.thrifty-treasurers" queue. The following configuratin for the finance-wizards queue applies this limit:
~~~
{
  "yarn.sheduler.capacity.root.finance-wizards.thrifty-treasures.minimum-user-limit-percent": "20"
}
~~~

The CapacityScheduler's leaf queues have the ability to restrict or expand a user's share within and beyond the queue's capacity through the per-leaf-queue "user-limit-factor" configuration. It denotes the fraction of queue capacity that any single user can grw, up to a maximum, irrespective of whether there are idle resources  in the cluster.
~~~
{
  "yarn.scheduler.capacity.root.finance-wizards.user-limit-factor": "1"
}
~~~

The default value of 1 means that any single user in that queue can, at maximum, occupy only the queue's configured capacity. This value avoids the case in which users in a single queue monopolize resources across all queues in a cluster. By extension, setting the value to 2 allows the queue to grow to a maximum of twice the size of the queue's configured capacity.

###### Reservations

The CapacityScheduler's responsibility is to match free resources in the cluster with the resource requirements of an application. Many times, however, a scheduling cycle occurs in such a way that even though there are free resources on a node, they are not large enough in size to satisfy the application that is at the head of the queue. This situation typically happens with large-memory applications whose resource demand for each of their containers is much larger than the typical application running in the cluster. When such applicaitons run in the cluster, anytime a regular applicaiton's containers finish, thereby releasing previously used resources for new cycles of scheduling, nodes will have freely available resources but the large-memory applications cannot take advantage of them because the resources are still too small. If left unchecked, this mismatch can cause starving of resource-intensive applications.

The CapacityScheduler solves this problem with a feature called reservation. The scheduling flow for reservations resembles the following:
- When a node reports in with a finished container and thus a certain amount of freely available resources, the scheduler chooses the right queue based on capacities and maximum capacities.

- Within that queue, the scheduler looks at the application in a FIFO order together with the user limits. Once a needy application is found, it tries to see if the requirements of that application can be met by this node's free capacity.

- If there is a size mismatch, the CapacityScheduler immediately creates a reservation for this applicaiton's container on this node.

- Once a reservation is made for an application on a node, those resources are not used by the scheduler for any other queue, application, or container until the original application for which the reservation was made is served.

- The node on which a reservation was made can eventually report back that enough containers have finished such that the total free capacity on the node now matches the reservation size. When that happends, the CapacityScheduler marks the reservation as fullfilled, removes it, and allocates a container on that node.

- Meanwhile, some other node may fullfill the resource needs of the application such that the application no longer needs the reserved capacity. In such a situation, when the reserved node eventually comes back, the reservation is simply cancelled.

The CapacityScheduler only maintains one active reservation per node.

###### State of the Queues

RUNNING(default) and STOPPED.

For an application to be accepted at any leaf queue, all of the queues in the ancestry - all the way to be root queue - need to be running.

The following configuration indicates the state of the 'finance-wizards' queue:
~~~
{
  "yarn.scheduler.capacity.root.finance-wizards.state": "RUNNING"
}
~~~

###### Limits on Applications

The following configuration controls the total number of concurrently active(both running and pending) applications at any single point in time:
~~~
{
  "yarn.scheduler.capacity.maximum-applications": "10000"
}
~~~

Every queue's maximum applications can be configured by:
~~~
{
  "yarn.scheduler.capacity.<queue-path>.maximum-applications": "absolute-capacity * yarn.scheduler.capacity.maximum-applications"
}
~~~

The limit on the maximum percentage of resources in the cluster that can be used by the ApplicationMasters can be configured by:
~~~
{
  "yarn.scheduler.capacity.maximum-am-resource-parent": "0.1"
}
~~~

For each queue can be configured by:
~~~
{
  "yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent": "0.1"
}
~~~
