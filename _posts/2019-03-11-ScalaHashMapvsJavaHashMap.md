---                                                 
layout: post  
title: Scala HashMap vs Java HashMap
date: 2019-03-11                                                  
author: AlstonWilliams                              
header-img: img/post-bg-2015.jpg   
catalog: true                                       
categories:                                         
- Scala                                    
tags:                                               
- Scala                                    
---

在这篇文章中，我们来探究一下Scala `HashMap.put(k, v)`以及Java `HashMap.put(k, v)`的性能。

具体代码没看，以后补充。

## Scala HashMap.put(k, v)

测试代码很简单:

~~~
package com.hyper

import java.util.concurrent.TimeUnit

import com.google.common.base.Stopwatch

import scala.collection.mutable

object TestScalaMap {

    def main(args: Array[String]): Unit = {

        val map: mutable.HashMap[String, String] = mutable.HashMap()

        val stopWatch = new Stopwatch()
        stopWatch.start()
        for (i <- 0 until 10000000) {
            map.put(i.toString, i.toString)
        }

        val elapse = stopWatch.elapsed(TimeUnit.SECONDS)

        println(elapse)

    }

}
~~~

运行结果是28s

## Java HashMap.put(k, v)

~~~
package com.hyper;

import com.google.common.base.Stopwatch;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class TestJavaMap {

    public static void main(String[] args) {
        Map<String, String> map = new HashMap<>();

        Stopwatch stopwatch = new Stopwatch();
        stopwatch.start();

        for (int i = 0; i < 10000000; i++) {
            map.put(String.valueOf(i), String.valueOf(i));
        }

        long elapse = stopwatch.elapsed(TimeUnit.SECONDS);
        System.out.println(elapse);
    }

}
~~~

运行结果是18s。
