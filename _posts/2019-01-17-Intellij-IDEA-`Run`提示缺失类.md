---
layout: post
title: Intellij-IDEA-`Run`提示缺失类
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 其它
tags:
- 其它
---
有时，我们明明在maven中明确写明了，有那么一个依赖，但是，在Run的时候，却会提示那个依赖中的Jar包找不到。比如:

~~~
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
~~~

但是，我们在Run的时候，这个包中的某些类就会找不到，我们查看运行的命令，也会找不到这个包。

在这种情况下，很好解决，就是直接把上面的**<scope>provided</scope>**去掉就好了。因为有这个标志的包，在Run的时候，不会加入到CLASSPATH中去。

然后再运行，我们会看到命令中已经包含这个依赖了。如果没有，那么你应该点击`File -> Invalidate caches and restart`，这下应该就可以了。
