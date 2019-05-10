---
layout: post
title: MapReduce读取HDFS上的文件时提示wrong fs
date: 2019-05-10
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Hadoop
tags:
- Hadoop
---

最近在部署客户环境时，遇到了这么一个问题
~~~
java.lang.illegalargumentexception wrong fs s3 // expected hdfs //
~~~

## 背景

这个客户的环境有点调皮，既使用了HDFS，又同时购买了S3。而且最蛋疼的是，他们没有把HDFS的defautFS设置成S3。

而我们的某个MR任务，运行时依赖的第三方Jar包，我们需要上传到HDFS上。然后在Driver中读取。

而我们如果上传到HDFS上，这个客户又总是会删掉这些Jar包。如果上传到S3上，又会遇到上面那个错误。

当然，有一个折衷的方案，就是每次跑任务的时候都把Jar包传上去。但是一旦客户把HDFS彻底删掉，就GG了。

而最诡异的是，如果我们在命令行中，通过`hadoop fs -put test s3://url//key`，是能传到S3上的。就是跑任务时，就算配置成S3上拿Jar包，还是会出错。

我们程序中的相关代码如下:
~~~
FileSystem fs = FileSystem.get(this.conf);
Utils.addFileToClassPath(conf, fs);
~~~

## 原因

这个问题看起来很诡异，其实还是蛮简单的。

我们先看一下，执行`hadoop fs -put`时，Hadoop是怎样确定应该将文件上传到哪个文件系统上的。

首先通过查看`hadoop`这个命令，确定`fs`子命令调用的是哪个类:
~~~
if [ "$COMMAND" = "fs" ] ; then
  CLASS=org.apache.hadoop.fs.FsShell
~~~

然后打开Hadoop的源码，找到**FsShell**,然后看到`-put`会调用**Put**这个类，然后找到其中的相关代码:
~~~
    @Override
    protected void processArguments(LinkedList<PathData> args)
    throws IOException {
      // NOTE: this logic should be better, mimics previous implementation
      if (args.size() == 1 && args.get(0).toString().equals("-")) {
        copyStreamToTarget(System.in, getTargetPath(args.get(0)));
        return;
      }
      super.processArguments(args);
    }
~~~

然后我们看下*copyStreamToTarget*的实现:
~~~
  protected void copyStreamToTarget(InputStream in, PathData target)
    throws IOException {
  if (target.exists && (target.stat.isDirectory() || !overwrite)) {
    throw new PathExistsException(target.toString());
  }
  TargetFileSystem targetFs = new TargetFileSystem(target.fs);
  try {
    PathData tempTarget = target.suffix("._COPYING_");
    targetFs.setWriteChecksum(writeChecksum);
    targetFs.writeStreamToFile(in, tempTarget, lazyPersist);
    targetFs.rename(tempTarget, target);
  } finally {
    targetFs.close(); // last ditch effort to ensure temp file is removed
  }
}
~~~

其中`target`就是`hadoop fs -put source dest`中的dest得到的一个数据结构。

我们大概能猜出来，就是*TargetFileSystem*决定了最终的存储位置。我们可以看到，它是通过`target.fs`的得到的。

那么,`target`是怎么来的呢？这里是重点。

它来自`CommandWithDestination.getRemoteDestination`:
~~~
protected void getRemoteDestination(LinkedList<String> args)
    throws IOException {
  if (args.size() < 2) {
    dst = new PathData(Path.CUR_DIR, getConf());
  } else {
    String pathString = args.removeLast();
    // if the path is a glob, then it must match one and only one path
    PathData[] items = PathData.expandAsGlob(pathString, getConf());
    switch (items.length) {
      case 0:
        throw new PathNotFoundException(pathString);
      case 1:
        dst = items[0];
        break;
      default:
        throw new PathIOException(pathString, "Too many matches");
    }
  }
}
~~~

其中调用的是`PathData.expandAsGlob`，往下层层扒开看，会看到它先解析了scheme，然后去找到**fs.{scheme}.impl**对应的类，然后实例化:
~~~
public static Class<? extends FileSystem> getFileSystemClass(String scheme,
    Configuration conf) throws IOException {
  if (!FILE_SYSTEMS_LOADED) {
    loadFileSystems();
  }
  Class<? extends FileSystem> clazz = null;
  if (conf != null) {
    clazz = (Class<? extends FileSystem>) conf.getClass("fs." + scheme + ".impl", null);
  }
  if (clazz == null) {
    clazz = SERVICE_FILE_SYSTEMS.get(scheme);
  }
  if (clazz == null) {
    throw new IOException("No FileSystem for scheme: " + scheme);
  }
  return clazz;
}
~~~

上面这段代码来自`FileSystem.getFileSystemClass`.

这里就很清楚了，之所以`hadoop fs -put`命令能推到S3上，是因为target中的scheme是"s3"，所以它初始化了对应的文件系统，然后传上去了。

所以我们客户的环境中，配置文件中，肯定是配置了这一点。我们打开他们的配置文件，果然找到了这一点:
~~~
<property>
  <name>fs.s3.impl</name>
  <value>com.amazon.ws.emr.hadoop.fs.EmrFileSystem</value>
<property>
~~~

好，那我们的代码会报错呢？

我们再贴一下我们的代码:
~~~
FileSystem fs = FileSystem.get(this.conf);
Utils.addFileToClassPath(conf, fs);
~~~

我们可以看到，第一行就已经确定了FileSystem。所以如果这儿初始化的FileSystem不是"com.amazon.ws.emr.hadoop.fs.EmrFileSystem"，那肯定就GG了。

我们来看看`FileSystem.get()`是怎样工作的:
~~~
/**
 * Returns the configured filesystem implementation.
 * @param conf the configuration to use
 */
public static FileSystem get(Configuration conf) throws IOException {
  return get(getDefaultUri(conf), conf);
}

/** Get the default filesystem URI from a configuration.
 * @param conf the configuration to use
 * @return the uri of the default filesystem
 */
public static URI getDefaultUri(Configuration conf) {
  return URI.create(fixName(conf.get(FS_DEFAULT_NAME_KEY, DEFAULT_FS)));
}
~~~

这就能说明一切了。

它是拿了配置文件中的**fs.defaultFS**。还记得前面背景中我们介绍的么？客户环境的**fs.defaultFS**被配置成了**hdfs://host:8020**.

所以这儿拿到的根本就不是"com.amazon.ws.emr.hadoop.fs.EmrFileSystem"，它肯定会GG呀。

## 解决方案

方案很简单，就是根据完全路径(如hdfs://host/something或者s3://host/something)来初始化FileSystem就好了.
