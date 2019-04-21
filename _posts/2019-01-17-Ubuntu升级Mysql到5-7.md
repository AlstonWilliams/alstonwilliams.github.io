---
layout: post
title: Ubuntu升级Mysql到5-7
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- MySQL
tags:
- MySQL
---
由于现在在做的一个项目中,需要全文检索的功能,而之前用的5.5.53版本的Mysql并没有实现全文检索的功能,在5.7中实现了,于是就考虑着升级到5.7版本.

我们这个只是一个小项目,目前只是起步阶段,还没有必要专门的使用ElasticSearch套装来做这一块.目前只使用Mysql5.7自带的全文检索功能,估计就完全能满足需求了.当然,在我们先实现了功能上的需求之后,日后会用ElasticSearch来改写它的.

那我们如何升级到Mysql5.7的呢?

从官网上的文档中,我们并不能找到明确的步骤,文档中更多的是告诉我们升级之前需要做的工作,以及为什么这么做,还有升级之后要做的工作,和升级可能出现的问题.

虽然并没有明确的步骤,但是我们还是建议各位读一下文档,提前了解一下风险.毕竟,如果是保存了很多关键数据库在升级过程中出现了问题,可就伤不起了.

首先,在升级前,我们需要先备份我们全部的数据库,执行下面这条命令:
**mysqldump --all-databases > all_databases.sql**

然后,我们还需要添加最新的APT包仓库.执行下面的命令,下载并执行包:
**wget https://dev.mysql.com/get/mysql-apt-config_0.8.1-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.1-1_all.deb**

在执行上面的那条*dpkg*的过程中,会弹出来一个窗口,问你要配置的Mysql产品.在其中的*Mysql Server*那一项里,选择**mysql-5.7**.然后选择**OK**.

![](http://upload-images.jianshu.io/upload_images/4108852-54c88bc87bbd3c36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后,更新包索引:
**sudo apt-get update**

然后,安装*MySQL-server*:
**sudo apt-get install mysql-server**

然后,升级全部的Mysql数据库:
**sudo mysql_upgrade -u root -p**

最后,重启mysql server:
**sudo service mysql restart**

需要注意的是,升级之后,你之前对Mysql所做的配置就没了.所以,需要重新配置一下.而在Mysql5.7中,选择了将配置文件分别存放在不同的目录中.关于Mysql5.7的配置文件的模板,请看这篇文章:
http://blog.programster.org/ubuntu-16-04-default-mysql-5-7-configuration/
