---
layout: post
title: docker-the-input-device-is-not-a-TTY
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
在cron中使用**docker exec -ti mysql mysqldump -A -u root -proot > /mysql_dump/all_dbs_$(date -I)**命令备份docker内的mysql数据库时，遇到的这个问题．

解决方法很简单，只需要去掉上面的命令中的**-ti**即可．
