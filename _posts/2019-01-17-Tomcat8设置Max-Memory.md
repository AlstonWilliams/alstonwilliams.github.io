---
layout: post
title: Tomcat8设置Max-Memory
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java
tags:
- Java
---
由于默认的Java默认的Max Memory是64M,有时这个太小，我们需要设置大一些的Max Memory,那我们该如何来做呢?

在tomcat的bin目录下，创建**setenv.sh**文件，同时添加如下内容:
**export CATALINA_OPTS="$CATALINA_OPTS -Xms512m"
export CATALINA_OPTS="$CATALINA_OPTS -Xmx8192m"
export CATALINA_OPTS="$CATALINA_OPTS -XX:MaxPermSize=256m"**

启动Tomcat,在启动日志中，我们就能看到变化．

如果你是从Ubuntu的源安装的Tomcat,那你可能会遇到一个问题，就是创建了上面的文件后，启动时从日志中还能看到又自动设置了别的*-Xms*和*-Xmx*等．这时，你需要将上面的内容修改成下面这样:
**export CATALINA_OPTS="-Xms512m"
export CATALINA_OPTS="$CATALINA_OPTS -Xmx8192m"
export CATALINA_OPTS="$CATALINA_OPTS -XX:MaxPermSize=256m"**

即开始设置**CATALINA_OPTS**这个变量时，直接就给它赋值．

我在调的时候就遇到了上面的那个问题，找了好长时间，把Tomcat的全部的文件都看遍了，也没找到自动设置别的*-Xms*和*-Xmx*的原因．根本就没有地方设置了*JAVA_OPTS*或者*CATALINA_OPTS*.但是启动的时候就是不对．

将文件内容设置成上面那样，解决!
