---
layout: post
title: Docker-seccomp
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 容器
tags:
- 容器
---
**注：本文翻译自[Docker Security – part 2](https://sreeninet.wordpress.com/2016/03/06/docker-security-part-2docker-engine/)中关于Seccomp的部分．请查看原文来获取更详细的信息．**

如果你用的是Ubuntu 14.04,并且是采用编译的方式来使用Docker,那么在你**docker run**一个容器时，你肯定会遇到这样一个错误:
**docker: Error response from daemon: Cannot start container daaa7a0c0b2c916019a68bbbdcc77e44a5b0a56478dc5c310665a00226923035: [9] System error: seccomp: config provided but seccomp not supported.
**

这个错误中提到了不支持**seccomp**.

那到底什么是**seccomp**呢？

**Seccomp**是**Secure computing mode**的缩写，它是Linux内核提供的一个操作，用于限制一个进程可以执行的系统调用．当然，我们需要有一个配置文件来指明进程到底可以执行哪些系统调用，不可以执行哪些系统调用．

在Docker中，它使用**Seccomp**来限制一个容器可以执行的系统调用．在Ubuntu14.04系统中，**default Docker binary**还不支持**Seccomp**．因此，我们要想使用**Seccomp**，就得使用**[static Docker binary](https://github.com/docker/docker/blob/master/docs/installation/binaries.m)**来安装Docker.在Ubuntu 14.x以后的版本中，**default Docker binary**中就默认支持**Seccomp**了．

默认情况下，**Seccomp**会禁止容器执行64位Linux系统的313个系统调用的44个．我找不到这个**Seccomp**的默认配置文件在哪．可能是写在了源码中．

我们下面来描述如何使用**Seccomp**．

首先，创建一个配置文件:**/home/smakam14/seccomp/profile.json**,其内容为:

~~~
{
    "defaultAction": "SCMP_ACT_ALLOW",
    "syscalls": [
        {
            "name": "chmod",
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
~~~

在上面的这个配置文件中，默认情况下，我们允许容器执行全部的系统调用．但是，禁止它执行**chmod**这个系统调用．

然后，我们用这个**Seccomp**配置文件来启动一个Docker容器:

**docker run --rm -it --security-opt seccomp:/home/smakam14/seccomp/profile.json busybox chmod 400 /etc/hosts**

结果如下:

**chmod: /etc/hosts: Operation not permitted**

我们使用**docker inspect**来查看容器的详细信息.会看到如下输出:
~~~
 "SecurityOpt": [
                "seccomp:     
                  {"defaultAction":"SCMP_ACT_ALLOW",
                  "syscalls":[{"name":"chmod","action":"SCMP_ACT_ERRNO"}]}"
            ],
~~~

我们也可以在**run**一个容器的时候，通过**--security-opt seccomp:unconfined**参数来允许容器执行全部的系统的调用:

**docker run --rm -it --security-opt seccomp:unconfined busybox chmod 400 /etc/hosts**
