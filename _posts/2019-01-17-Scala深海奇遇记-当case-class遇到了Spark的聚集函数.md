---
layout: post
title: Scala深海奇遇记-当case-class遇到了Spark的聚集函数
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---
自从知道有case class这个东西以后，一直都比较常用这个东西。但是，最近在测试的时候，突然发现，其实这个东西并不简单，它导致了一个看起来很无厘头的错误，并且花了我两天的时间来调试。

在这篇文章里，我会详细记录调试的过程，以及结论。

## 致谢

在调试的过程中，得到了我们Hadoop组老大，项目组老大，以及其他同事的深度支持与帮助，非常感谢他们。

## 结论

先说结论。如果有朋友不感兴趣，不想深究原理，只是想知道怎么用，可以跳过后面的部分。

不要把case class放在class里面。例如:
~~~
class Test {

  case class A(a: String)

}
~~~

可以放到object里面，或者放到package里面。例如:
~~~
object Test {

  case class A(a: String)

}
~~~

或者

~~~
package Test {

  case class A(a: String)

}
~~~

如果不遵循这条原则，那么有极大的概率，Spark在执行的时候，相同的key并不会聚集到一起。

## 分析

首先，我们有如下代码:

~~~
package org.apache.spark.test

import org.apache.spark.{Partitioner, SparkConf, SparkContext}

import scala.util.Random

object VerifySparkBug {

  def main(args: Array[String]): Unit = {
    new VerifySparkBug().run()
  }

}

class VerifySparkBug extends Serializable {

  def run() = {

    val sparkConf = new SparkConf()
    sparkConf.setMaster("local")
    sparkConf.setAppName("VerifySparkBug")
    val sparkContext = new SparkContext(sparkConf)

    val inputRDD = sparkContext.parallelize(Seq(
      (UidAndPartition("1", "2"), 1L),
      (UidAndPartition("1", "1"), 1L),
      (UidAndPartition("1", "3"), 1L),
      (UidAndPartition("2", "1"), 1L),
      (UidAndPartition("2", "2"), 1L),
      (UidAndPartition("3", "3"), 1L)
    )).partitionBy(new RandomPartitioner(10))

    inputRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ inputRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val uidRDD = inputRDD.map{
      case pair => {
        (Uid(pair._1.uid), pair._2)
      }
    }

    uidRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ uidRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val result = uidRDD.aggregateByKey(0L)(
      (counter: Long, number: Long) => {
        counter + number
      },
      (counter1: Long, counter2: Long) => {
        counter1 + counter2
      }
    ).collect()

    for (pair <- result) {
      System.out.println("------ key: " + pair._1 + ", uv: " + pair._2)
    }

  }

  case class UidAndPartition(uid: String, partition: String)

  case class Uid(idType: String)
}

class RandomPartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions

  override def getPartition(key: Any): Int = {
    new Random().nextInt(partitions)
  }
}
~~~

它的输出如下:
~~~
------ key: Uid(1), uv: 1
------ key: Uid(1), uv: 1
------ key: Uid(1), uv: 1
------ key: Uid(3), uv: 1
------ key: Uid(2), uv: 1
------ key: Uid(2), uv: 1
~~~

很诡异，对吧？明明key相同，却并没有被聚合到一起。而且，更诡异的是，key相同的hash值也相同。

#### 猜想1

开始，我们怀疑是case class虽然默认实现了**Serializable**接口，但是并没有生成**serialVersionUID**，所以在序列化以及反序列化时，生成的类其实并不是同一个。导致反序列化出来的对象不是同一个。

反编译后的*Uid*的部分代码如下:

![](https://upload-images.jianshu.io/upload_images/4108852-e7028a73ff02f488.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


于是，我就用Netty写了一个服务器端和客户端的程序，来测试这个问题:

服务器端:
~~~
package org.apache.spark.test

import java.io._

import io.netty.bootstrap.ServerBootstrap
import io.netty.buffer.ByteBuf
import io.netty.channel.{ChannelHandlerContext, ChannelInboundHandlerAdapter, ChannelInitializer, SimpleChannelInboundHandler}
import io.netty.channel.nio.NioEventLoopGroup
import io.netty.channel.socket.nio.{NioServerSocketChannel, NioSocketChannel}
import org.apache.spark.test.VerifySparkBug.Uid

object ServerForCaseClassEquals {

  def main(args: Array[String]): Unit = {

    val serverBootstrap = new ServerBootstrap()
    val bossEventLoopGroup = new NioEventLoopGroup()
    val workerEventLoopGroup = new NioEventLoopGroup()

    serverBootstrap.group(bossEventLoopGroup, workerEventLoopGroup)
      .channel(classOf[NioServerSocketChannel])
      .childHandler(new ChannelInitializer[NioSocketChannel] {
        override def initChannel(c: NioSocketChannel) = {
          c.pipeline().addLast(new ChannelInboundHandlerAdapter {
            override def channelRead(ctx: ChannelHandlerContext, msg: scala.Any): Unit = {
              val byteBuf = msg.asInstanceOf[ByteBuf]
              val length = byteBuf.readInt()
              val bytes = new Array[Byte](length)
              byteBuf.readBytes(bytes)

              val byteInputStream = new ByteArrayInputStream(bytes)
              val objectInputStream = new ObjectInputStream(byteInputStream)

              println(objectInputStream.readObject() == Uid("a"))

            }
          })
        }
      })
      .bind(8000)

  }

}
~~~

客户端:
~~~
package org.apache.spark.test

import java.io.{ByteArrayOutputStream, ObjectOutputStream}

import io.netty.bootstrap.Bootstrap
import io.netty.channel.nio.NioEventLoopGroup
import io.netty.channel.socket.nio.NioSocketChannel
import io.netty.channel.{ChannelHandlerContext, ChannelInboundHandlerAdapter, ChannelInitializer}
import org.apache.spark.test.VerifySparkBug.Uid

object ClientForCaseClassEquals {

  def main(args: Array[String]): Unit = {

    val bootstrap = new Bootstrap()
    val eventLoopGroup = new NioEventLoopGroup()

    bootstrap.group(eventLoopGroup)
      .channel(classOf[NioSocketChannel])
      .handler(new ChannelInitializer[NioSocketChannel] {
        override def initChannel(c: NioSocketChannel) = {
          c.pipeline().addLast(new ChannelInboundHandlerAdapter {
            override def channelActive(ctx: ChannelHandlerContext): Unit = {
              val uid = Uid("a")
              val byteArrayOutputStream = new ByteArrayOutputStream()
              val objectOutputStream = new ObjectOutputStream(byteArrayOutputStream)

              objectOutputStream.writeObject(uid)

              objectOutputStream.close()

              val bytes = byteArrayOutputStream.toByteArray

              val byteBuf = ctx.alloc().buffer()
              byteBuf.writeInt(bytes.length)
              byteBuf.writeBytes(bytes)

              ctx.channel().writeAndFlush(byteBuf)
            }
          })
        }
      })
      .connect("localhost", 8000)

  }

}
~~~

而这个测试自然失败了。测试的结果是，反序列化解析出来的对象，是**==**反序列化之前的对象的。

为什么用**==**？因为在scala中，**==**其实就是**equals()**方法的语法糖。而且，spark的代码中，聚合函数中，对key进行比较时，也是用的**==**。

其实这种假设根本就经不起推敲。

如果序列化和反序列的时候，类不是一个，即*serialVersionUID*不同，那么，其实反序列化是不会成功的，它是会直接报错的。而我们的程序中并没有报错。说明其实是同一个。

那么既然case class在生成的时候，并没有生成*serialVersionUID*。那我们怎么能够确定，在运行的时候，即使是在不同的节点，不同的JVM上，它们就是同一个呢？这个问题，在Java Specification中的序列化和反序列化部分有详细描述。总之，就是根据类的相关信息，生成一个*serialVersionUID*。由于这个信息跟内存地址无关，所以即使是运行时生成，也是唯一且相同的。

#### 猜想2

这个假设给推掉了。为了复现这个问题，我写了如下代码，打算来进行复现。上面的代码是最终版的复现版本。在下面的代码中，虽然问题看似得到了复现，实际上是完全不同的问题。

~~~
package org.apache.spark.test

import org.apache.spark.{Partitioner, SparkConf, SparkContext}

import scala.util.Random

object VerifySparkBug {

  def main(args: Array[String]): Unit = {
    new VerifySparkBug().run()
  }

}

class VerifySparkBug extends Serializable {

  def run() = {

    val sparkConf = new SparkConf()
    sparkConf.setMaster("local")
    sparkConf.setAppName("VerifySparkBug")
    val sparkContext = new SparkContext(sparkConf)

    val inputRDD = sparkContext.parallelize(Seq(
      (UidAndPartition("1", "2"), 1L),
      (UidAndPartition("1", "1"), 1L),
      (UidAndPartition("1", "3"), 1L),
      (UidAndPartition("2", "1"), 1L),
      (UidAndPartition("2", "2"), 1L),
      (UidAndPartition("3", "3"), 1L)
    ))

    inputRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ inputRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val uidRDD = inputRDD.map{
      case pair => {
        (Uid(pair._1.uid), pair._2)
      }
    }.partitionBy(new RandomPartitioner(10))

    uidRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ uidRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val result = uidRDD.aggregateByKey(0L)(
      (counter: Long, number: Long) => {
        counter + number
      },
      (counter1: Long, counter2: Long) => {
        counter1 + counter2
      }
    ).collect()

    for (pair <- result) {
      System.out.println("------ key: " + pair._1 + ", uv: " + pair._2)
    }

  }

  case class UidAndPartition(uid: String, partition: String)

  case class Uid(idType: String)
}

class RandomPartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions

  override def getPartition(key: Any): Int = {
    new Random().nextInt(partitions)
  }
}
~~~

代码跟上面的很相似，只是*partitionBy()*的位置变了一下。

初衷很简单，因为我们并不能保证相同的*Uid*会在同一个partition，于是就通过*RandomPartitioner*来模拟随机分区的效果。

这段代码的问题是，如果在进行聚集(如aggregateByKey, reduceByKey等)之前，你有进行过*partitionBy*操作，那么，Spark会认为，你已经将相同的Key放到一个分区了，所以它在进行聚集的时候，是不会考虑聚集不同partition上，相同的key的情况的。

我们通过调试聚集函数通用的*PairRDDFunctions.combineByKeyWithClassTag*方法，就能看到这个结论:

![](https://upload-images.jianshu.io/upload_images/4108852-115d4503bb10f07e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到，如果聚集函数前面刚好调用了*partitionBy*方法，聚集函数内部是不会调用*partitionBy*来将相同的Key分到一个分区的。

这是因为*partitionBy*以及聚集函数，其实都是*ShuffleDependency*。Spark内部做了优化，如果*ShuffleDependency*前面已经是*Narrow Dependency*了，那么就会把这个*ShuffleDependency*转换成*Narrow Dependency*。

所以上面看似正确的复现，实际上刚好走进了这个陷阱里面。

那么，上面的代码，如果考虑到这个问题，该怎么解决？一种方案是，通过**HashPartitioner**，将相同的key确实分到同一个分区里面去。另一种方法就是，在上面的代码中的**partitionBy(new RandomPartitioner(10))** 后面，加一个**map(r => r)**。 为什么加个这个就会生效?

我们先看**map()** 函数，以及**MapPartitionRDD**的代码。

![](https://upload-images.jianshu.io/upload_images/4108852-a02f4243118de56e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/4108852-4716a4d5abfef63d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到,**map()**函数，实际上是生成了一个没有partitioner的**MapPartitionRDD**。就是这个没有partitioner，帮了我们的大忙。没有partitioner，聚集函数就不知道前面是*Narrow Dependency*，所以就会考虑相同的key在不同分区的问题。

下面这张图片，是我采用第二种方式，调试的过程。可以看到，现在走的是**ShuffleRDD**。

![](https://upload-images.jianshu.io/upload_images/4108852-75af602d9715e4d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后，我们可以看到，丫的，相同的key还是没有被聚集到一起。

其实当时在写这个复现的Demo的时候，是聚到一起了，一度手舞足蹈。但是聚到一起的原因，是case class是放在Object里面了，没有放到class里，所以并没有正确的复现问题。

#### 猜想3

采用猜想2中的第1种方法，我们把不同分区中，相同的Key放到同一个分区里，有如下代码:

~~~
package org.apache.spark.test

import org.apache.spark.{HashPartitioner, Partitioner, SparkConf, SparkContext}

import scala.util.Random

object VerifySparkBug {

  def main(args: Array[String]): Unit = {
    new VerifySparkBug().run()
  }

}

class VerifySparkBug extends Serializable {

  def run() = {

    val sparkConf = new SparkConf()
    sparkConf.setMaster("local")
    sparkConf.setAppName("VerifySparkBug")
    val sparkContext = new SparkContext(sparkConf)

    val inputRDD = sparkContext.parallelize(Seq(
      (UidAndPartition("1", "2"), 1L),
      (UidAndPartition("1", "1"), 1L),
      (UidAndPartition("1", "3"), 1L),
      (UidAndPartition("2", "1"), 1L),
      (UidAndPartition("2", "2"), 1L),
      (UidAndPartition("3", "3"), 1L)
    ))

    inputRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ inputRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val uidRDD = inputRDD.map{
      case pair => {
        (Uid(pair._1.uid), pair._2)
      }
    }.partitionBy(new RandomPartitioner(10)).partitionBy(new HashPartitioner(10))

    uidRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ uidRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val result = uidRDD.aggregateByKey(0L)(
      (counter: Long, number: Long) => {
        counter + number
      },
      (counter1: Long, counter2: Long) => {
        counter1 + counter2
      }
    ).collect()

    for (pair <- result) {
      System.out.println("------ key: " + pair._1 + ", uv: " + pair._2)
    }

  }

  case class UidAndPartition(uid: String, partition: String)

  case class Uid(idType: String)
}

class RandomPartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions

  override def getPartition(key: Any): Int = {
    new Random().nextInt(partitions)
  }
}
~~~

其实这些代码差距很小，用文本比较工具，很容易看出差距。

在运行这个过程中，我们发现，丫的，把它们放到同一个分区，还是不能聚到一起？太他么过分了。

于是，我们查看聚集函数中，是如何比较key的。

![](https://upload-images.jianshu.io/upload_images/4108852-81c7ea011a94453d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，是调用**equals**方法。

于是，我们就想，case class的**equals**方法，到底长什么样子？

这里有一点很坑，就是你看反编译出来的代码，在这里是会反编译错误的。你必须找其他的方式来看。我们采用的方式，是看Scala编译的中间结果。

运行**scalac -Xprint:typer VerifySparkBug.scala**，我们能看到，上面的代码中，**Uid的equals()**方法，scala会被编译成这种形式:

~~~
override <synthetic> def equals(x$1: Any): Boolean = Uid.this.eq(x$1.asInstanceOf[Object]).||(x$1 match {
  case (_: VerifySparkBug.this.Uid) => true
  case _ => false
}.&&({
        <synthetic> val Uid$1: VerifySparkBug.this.Uid = x$1.asInstanceOf[VerifySparkBug.this.Uid];
        Uid.this.idType.==(Uid$1.idType).&&(Uid$1.canEqual(Uid.this))
      }))
    };
~~~

我们再来看看，把Uid放到Object里面，生成的**equals**方法是什么样子。

代码如下:
~~~
package org.apache.spark.test

import org.apache.spark.test.VerifySparkBug.{Uid, UidAndPartition}
import org.apache.spark.{HashPartitioner, Partitioner, SparkConf, SparkContext}

import scala.util.Random

object VerifySparkBug {

  def main(args: Array[String]): Unit = {
    new VerifySparkBug().run()
  }

  case class UidAndPartition(uid: String, partition: String)

  case class Uid(idType: String)
}

class VerifySparkBug extends Serializable {

  def run() = {

    val sparkConf = new SparkConf()
    sparkConf.setMaster("local")
    sparkConf.setAppName("VerifySparkBug")
    val sparkContext = new SparkContext(sparkConf)

    val inputRDD = sparkContext.parallelize(Seq(
      (UidAndPartition("1", "2"), 1L),
      (UidAndPartition("1", "1"), 1L),
      (UidAndPartition("1", "3"), 1L),
      (UidAndPartition("2", "1"), 1L),
      (UidAndPartition("2", "2"), 1L),
      (UidAndPartition("3", "3"), 1L)
    ))

    inputRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ inputRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val uidRDD = inputRDD.map{
      case pair => {
        (Uid(pair._1.uid), pair._2)
      }
    }.partitionBy(new RandomPartitioner(10)).partitionBy(new HashPartitioner(10))

    uidRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ uidRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val result = uidRDD.aggregateByKey(0L)(
      (counter: Long, number: Long) => {
        counter + number
      },
      (counter1: Long, counter2: Long) => {
        counter1 + counter2
      }
    ).collect()

    for (pair <- result) {
      System.out.println("------ key: " + pair._1 + ", uv: " + pair._2)
    }

  }

}

class RandomPartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions

  override def getPartition(key: Any): Int = {
    new Random().nextInt(partitions)
  }
}
~~~

生成的**equals()**方法如下:

~~~
override <synthetic> def equals(x$1: Any): Boolean = Uid.this.eq(x$1.asInstanceOf[Object]).||(x$1 match {
  case (_: org.apache.spark.test.VerifySparkBug.Uid) => true
  case _ => false
}.&&({
        <synthetic> val Uid$1: org.apache.spark.test.VerifySparkBug.Uid = x$1.asInstanceOf[org.apache.spark.test.VerifySparkBug.Uid];
        Uid.this.idType.==(Uid$1.idType).&&(Uid$1.canEqual(Uid.this))
      }))
    };
~~~

我们可以看到，它们的区别，就在于放在class中的Uid,调用的是**VerifySparkBug.this.Uid**。而放在object中的Uid,调用的是**com.hyper.cdp.label.spark.test.VerifySparkBug.Uid**。后者很容易理解，它肯定是唯一且相同的。但是前者呢？它包含了一个**VerifySparkBug.this**啊。

那么什么是**VerifySparkBug.this**呢？我也不清楚。但是我写了一个Demo，验证了一个事实，对于每一个**VerifySparkBug**实例，它们的值都是不同的。

代码如下:

~~~
package org.apache.spark.test

class VerifyCaseClassInClass {

    case class Test(a: String)

    def test() = {
        println(VerifyCaseClassInClass.this)
    }
}

object VerifyCaseClassInClass {

    def main(args: Array[String]): Unit = {

        println(new VerifyCaseClassInClass().Test("a").equals(new VerifyCaseClassInClass().Test("a")))

        new VerifyCaseClassInClass().test()
        new VerifyCaseClassInClass().test()

    }

}
~~~

这就很容易解释为什么case class在Class中时，即使Key相同，仍然不会不会聚到一起了。

在Shuffle的时候，对于收到的其他分区的Key，肯定是为每个分区都new一个**VerifySparkBug**。这样就导致看起来相同的key，即使它们的hash值也一样，但是就是聚不到一起。

case class的*hashCode()*方法倒是很中立，它跟内存地址等都无关。只要字段的值一样，生成的hash就相同。感兴趣的读者可以自行查看**scala.runtime.ScalaRunTime._hashCode**这个方法。

这次调试，也深刻验证了，两个对象的hashcode相同，它们却不一定equals这一个简单却并没有重视过的问题。

最终，能够正常工作的代码如下:

~~~
package org.apache.spark.test

import org.apache.spark.test.VerifySparkBug.{Uid, UidAndPartition}
import org.apache.spark.{HashPartitioner, Partitioner, SparkConf, SparkContext}

import scala.util.Random

object VerifySparkBug {

  def main(args: Array[String]): Unit = {
    new VerifySparkBug().run()
  }

  case class UidAndPartition(uid: String, partition: String)

  case class Uid(idType: String)
}

class VerifySparkBug extends Serializable {

  def run() = {

    val sparkConf = new SparkConf()
    sparkConf.setMaster("local")
    sparkConf.setAppName("VerifySparkBug")
    val sparkContext = new SparkContext(sparkConf)

    val inputRDD = sparkContext.parallelize(Seq(
      (UidAndPartition("1", "2"), 1L),
      (UidAndPartition("1", "1"), 1L),
      (UidAndPartition("1", "3"), 1L),
      (UidAndPartition("2", "1"), 1L),
      (UidAndPartition("2", "2"), 1L),
      (UidAndPartition("3", "3"), 1L)
    ))

    inputRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ inputRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val uidRDD = inputRDD.map{
      case pair => {
        (Uid(pair._1.uid), pair._2)
      }
    }.partitionBy(new RandomPartitioner(10)).partitionBy(new HashPartitioner(10))

    uidRDD.foreachPartition {
      case iterator => {
        val partitionId = new Random().nextInt()
        while (iterator.hasNext) {
          val item = iterator.next()
          System.out.println("------ uidRDD partitionId: " + partitionId
            + ", key: " + item._1
            + ", hash:" + item._1.hashCode()
            + ",value: " + item._2)
        }
      }
    }

    val result = uidRDD.aggregateByKey(0L)(
      (counter: Long, number: Long) => {
        counter + number
      },
      (counter1: Long, counter2: Long) => {
        counter1 + counter2
      }
    ).collect()

    for (pair <- result) {
      System.out.println("------ key: " + pair._1 + ", uv: " + pair._2)
    }

  }

}

class RandomPartitioner(partitions: Int) extends Partitioner {
  override def numPartitions: Int = partitions

  override def getPartition(key: Any): Int = {
    new Random().nextInt(partitions)
  }
}
~~~
