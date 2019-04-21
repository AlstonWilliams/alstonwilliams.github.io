---
layout: post
title: send-mail--fatal--parameter-inet_interfaces--no-local-interface-found
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
当Cron执行命令出错时，默认会发送邮件给cron任务的所有者．当然，发送邮件时也可能会出错．我就遇到了如题所示的错误．

我使用的机器环境为Centos7.它默认会使用Postfix来发送邮件．

我们找到其配置文件**/etc/postfix/main.cf**，会看到其中有如下一行:
**inet_interfaces = localhost**

将其修改为**inet_interfaces = all**，然后重启一下服务就可以了．
