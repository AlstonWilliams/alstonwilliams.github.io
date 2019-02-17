---
layout: post
title: Java-NIO-SelectorProvider与IO多路复用
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- JDK源码研读
tags:
- JDK源码研读
---
最近在学习Netty,看了好多资料,也看了一部分<<Netty in action>>这本书,发现完全不能理解它的设计,它的组件.

联想到Netty主要是一个NIO框架,于是觉得是因为对NIO的了解不够而导致的.然后就查阅NIO的相关资料,发现还是不能理解其原理.

不得不说,Google了很多资料,包括英文的和中文的,大多数都是NIO的具体用法,而对于其核心组件,比如selector,他们的作用,实现原理,却并没有说明.看完网上的介绍之后,让我更加懵懵哒了.

既然查询不到结果,就想自己查看源码了解其原理.于是查看了Oracle JDK1.8.0_91中和NIO相关的部分的源码,以及openjdk1.7的部分源码.因为Oracle JDK1.8.0_91中,对于一些类的实现,并没有给出,只是给出的.class文件.即使我们可以通过反编译来获得,但是终究还是太麻烦.所以这部分源码,就从openjdk1.7来获得.

## Java NIO SelectorProvider

查看Oracle JDK1.8.0_91源码时,我们可以看到**Selector**这个组件,是由**SelectorProvider**创建的.


![](http://upload-images.jianshu.io/upload_images/4108852-1e0a61939a3ed322.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们看一下**SelectorProvider.provider()**方法的具体实现:


![](http://upload-images.jianshu.io/upload_images/4108852-85ac758e3ecaa710.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


查看**loadProviderFromProperty()**方法和**loadProviderAsService()**方法的源码:


![](http://upload-images.jianshu.io/upload_images/4108852-1cb9b777fdcadf61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到,**SelectorProvider.provider()**方法会在**System Property**中不存在**java.nio.channels.spi.SelectorProvider**属性和不能找到**SelectorProvider**的实现类时,创建一个默认的**sun.nio.ch.DefaultSelectorProvider**来作为**SelectorProvider**.

我们从open jdk7中查看**sun.nio.ch.DefaultSelectorProvider**的源码:


![](http://upload-images.jianshu.io/upload_images/4108852-ad38859eca05267f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


open jdk7的源码中,提供了三个版本的**sun.nio.ch.DefaultSelectorProvider**的实现:


![](http://upload-images.jianshu.io/upload_images/4108852-9deff58f06357930.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们这里选择的是**solaris**版本的.

从**sun.nio.ch.DefaultSelectorProvider**的源码中,我们可以看到,如果是**linux**机器,并且其内核版本大于**2.6**,创建的就是**EPollSelectorProvider**,否则的话,就创建**PollSelectorProvider**.

这就是我们今天要介绍的重点-IO多路复用.

## IO多路复用

IO多路复用就是我们说的select,poll, epoll,接下来我们会逐个介绍.

### Select

#### 基本概念
IO多路复用是指内核一旦发现进程指定的一个或者多个IO条件准备读取，它就通知该进程。IO多路复用适用如下场合：

- 当客户处理多个描述字时（一般是交互式输入和网络套接口），必须使用I/O复用。

- 当一个客户同时处理多个套接口时，而这种情况是可能的，但很少出现。

- 如果一个TCP服务器既要处理监听套接口，又要处理已连接套接口，一般也要用到I/O复用。

- 如果一个服务器即要处理TCP，又要处理UDP，一般要使用I/O复用。

- 如果一个服务器要处理多个服务或多个协议，一般要使用I/O复用。

与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

#### select函数
该函数准许进程指示内核等待多个事件中的任何一个发送，并只在有一个或多个事件发生或经历一段指定的时间后才唤醒。函数原型如下：

**int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)**

函数参数介绍如下：

（1）第一个参数maxfdp1指定待测试的描述字个数，它的值是待测试的最大描述字加1（因此把该参数命名为maxfdp1），描述字0、1、2...maxfdp1-1均将被测试。因为文件描述符是从0开始的。

（2）中间的三个参数readset、writeset和exceptset指定我们要让内核测试读、写和异常条件的描述字。如果对某一个的条件不感兴趣，就可以把它设为空指针。struct fd_set可以理解为一个集合，这个集合中存放的是文件描述符，可通过以下四个宏进行设置：
~~~
          void FD_ZERO(fd_set *fdset);           //清空集合

          void FD_SET(int fd, fd_set *fdset);   //将一个给定的文件描述符加入集合之中

          void FD_CLR(int fd, fd_set *fdset);   //将一个给定的文件描述符从集合中删除

          int FD_ISSET(int fd, fd_set *fdset);   // 检查集合中指定的文件描述符是否可以读写 
~~~
（3）timeout告知内核等待所指定描述字中的任何一个就绪可花多少时间。其timeval结构用于指定这段时间的秒数和微秒数。
~~~
         struct timeval{

                   long tv_sec;   //seconds

                   long tv_usec;  //microseconds

       };
~~~
这个参数有三种可能：

（1）永远等待下去：仅在有一个描述字准备好I/O时才返回。为此，把该参数设置为空指针NULL。

（2）等待一段固定时间：在有一个描述字准备好I/O时返回，但是不超过由该参数所指向的timeval结构中指定的秒数和微秒数。

（3）根本不等待：检查描述字后立即返回，这称为轮询。为此，该参数必须指向一个timeval结构，而且其中的定时器值必须为0。

#### 基本原理图
![](http://upload-images.jianshu.io/upload_images/4108852-5c78e26976133281.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### poll

#### 基本知识
poll的机制与select类似，与select在本质上没有多大差别，管理多个描述符也是进行轮询，根据描述符的状态进行处理，但是poll没有最大文件描述符数量的限制。poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

#### poll函数
**int poll ( struct pollfd * fds, unsigned int nfds, int timeout);**

pollfd结构体定义如下：

~~~
struct pollfd {

int fd;         /* 文件描述符 */
short events;         /* 等待的事件 */
short revents;       /* 实际发生了的事件 */
} ; 
~~~

每一个pollfd结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示poll()监视多个文件描述符。每个结构体的events域是监视该文件描述符的事件掩码，由用户来设置这个域。revents域是文件描述符的操作结果事件掩码，内核在调用返回时设置这个域。events域中请求的任何事件都可能在revents域中返回。合法的事件如下：

~~~
　　POLLIN 　　　　　　　　有数据可读。

　　POLLRDNORM 　　　　  有普通数据可读。

　　POLLRDBAND　　　　　 有优先数据可读。

　　POLLPRI　　　　　　　　 有紧迫数据可读。

　　POLLOUT　　　　　　      写数据不会导致阻塞。

　　POLLWRNORM　　　　　  写普通数据不会导致阻塞。

　　POLLWRBAND　　　　　   写优先数据不会导致阻塞。

　　POLLMSGSIGPOLL 　　　　消息可用。

　　此外，revents域中还可能返回下列事件：
　　POLLER　　   指定的文件描述符发生错误。

　　POLLHUP　　 指定的文件描述符挂起事件。

　　POLLNVAL　　指定的文件描述符非法。
~~~

这些事件在events域中无意义，因为它们在合适的时候总是会从revents中返回。

使用poll()和select()不一样，你不需要显式地请求异常情况报告。

POLLIN | POLLPRI等价于select()的读事件，POLLOUT |POLLWRBAND等价于select()的写事件。POLLIN等价于POLLRDNORM |POLLRDBAND，而POLLOUT则等价于POLLWRNORM。例如，要同时监视一个文件描述符是否可读和可写，我们可以设置 events为POLLIN |POLLOUT。在poll返回时，我们可以检查revents中的标志，对应于文件描述符请求的events结构体。如果POLLIN事件被设置，则文件描述符可以被读取而不阻塞。如果POLLOUT被设置，则文件描述符可以写入而不导致阻塞。这些标志并不是互斥的：它们可能被同时设置，表示这个文件描述符的读取和写入操作都会正常返回而不阻塞。

timeout参数指定等待的毫秒数，无论I/O是否准备好，poll都会返回。timeout指定为负数值表示无限超时，使poll()一直挂起直到一个指定事件发生；timeout为0指示poll调用立即返回并列出准备好I/O的文件描述符，但并不等待其它的事件。这种情况下，poll()就像它的名字那样，一旦选举出来，立即返回。

成功时，poll()返回结构体中revents域不为0的文件描述符个数；如果在超时前没有任何事件发生，poll()返回0；失败时，poll()返回-1，并设置errno为下列值之一：
~~~
　　EBADF　　       一个或多个结构体中指定的文件描述符无效。

　　EFAULTfds　　 指针指向的地址超出进程的地址空间。

　　EINTR　　　　  请求的事件之前产生一个信号，调用可以重新发起。

　　EINVALnfds　　参数超出PLIMIT_NOFILE值。

　　ENOMEM　　     可用内存不足，无法完成请求。
~~~

### epoll

#### 基本知识

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

#### epoll接口

epoll操作过程需要三个接口，分别如下：

~~~
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
~~~

（1） int epoll_create(int size);
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大。这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值。需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

（2）int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll_create()的返回值，第二个参数表示动作，用三个宏来表示：

~~~
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
~~~

第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：
~~~
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
~~~

events可以是以下几个宏的集合：
~~~
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
~~~
（3） int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
　　等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。
  
  
#### 工作模式

epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

**LT模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

**ET模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。


## 参考资料
[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859#articleHeader8)
[IO多路复用之select总结](http://www.cnblogs.com/Anker/archive/2013/08/14/3258674.html)
[IO多路复用之poll总结](http://www.cnblogs.com/Anker/archive/2013/08/15/3261006.html)
[IO多路复用之epoll总结](http://www.cnblogs.com/Anker/archive/2013/08/17/3263780.html)
