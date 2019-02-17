---
layout: post
title: Java三个线程分别打印十次A,B,C，要求打印出ABCABC----的形式
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Java疑难问题
tags:
- Java疑难问题
---
今天一位同学问我题目中的这个问题，并给了我下面的代码，花了好久才看懂，这里总结一下．

实现代码如下所示：

~~~~
package com.multithread.wait;  
public class MyThreadPrinter2 implements Runnable {     
        
    private String name;     
    private Object prev;     
    private Object self;     
    
    private MyThreadPrinter2(String name, Object prev, Object self) {     
        this.name = name;     
        this.prev = prev;     
        this.self = self;     
    }     
    
    @Override    
    public void run() {     
        int count = 10;     
        while (count > 0) {     
            synchronized (prev) {     
                synchronized (self) {     
                    System.out.print(name);     
                    count--;    
                      
                    self.notify();     
                }     
                try {     
                    prev.wait();     
                } catch (InterruptedException e) {     
                    e.printStackTrace();     
                }     
            }     
    
        }     
    }     
    
    public static void main(String[] args) throws Exception {     
        Object a = new Object();     
        Object b = new Object();     
        Object c = new Object();     
        MyThreadPrinter2 pa = new MyThreadPrinter2("A", c, a);     
        MyThreadPrinter2 pb = new MyThreadPrinter2("B", a, b);     
        MyThreadPrinter2 pc = new MyThreadPrinter2("C", b, c);     
             
             
        new Thread(pa).start();  
        Thread.sleep(100);  //确保按顺序A、B、C执行  
        new Thread(pb).start();  
        Thread.sleep(100);    
        new Thread(pc).start();     
        Thread.sleep(100);    
        }     
}
~~~~

首先，我们提出一个概念模型，就是Java对象中，包含了这么两部分内容：

**ArrayList<Thread> threadsWhoWaitForThisObject
Thread threadWhoEnterTheMonitor**

在执行synchronized块时，线程会比较相应对象的**threadWhoEnterTheMonitor**是否是null或者当前线程，如果是的话，就将相应对象的**threadWhoEnterTheMonitor**设置为当前线程，并且执行synchronized块．当线程调用wait()方法时，会将线程加入到对应对象的**threadsWhoWaitForThisObject**，并且将对应对象的**threadWhoEnterTheMonitor**设置为null，让其他的线程可以得到执行．当线程从synchronized块中出来时，也会将对应对象的**threadsWhoEnterTheMonitor**设置为null.

当然，这只是一个概念模型．Java中对象是否包含上面提到的这两个区域，我还不清楚．

清楚了上面的概念模型后，我们照着上面的代码走一个循环就清楚了．

首先，是线程pa执行，其次是pb，最后是线程pc执行．同时有三个对象，a，b，c，它们和上面的三个线程是一一对应的关系．下面我们将会从这三个执行期间a，b，c中的**threadWhoEnterTheMonitor**以及**threadsWhoWaitForThisObject**的变化来解释其实现原理．

在第一次pa执行期间，三个对象的**threadWhoEnterTheMonitor**以及**threadsWhoWaitForThisObject**的变化如下：
~~~~
c's threadWhoEnterTheMonitor: pa -> null
a's threadWhoEnterTheMonitor: pa -> null
b's threadWhoEnterTheMonitor: null

a's threadsWhoWaitForThisObject: null
b's threadsWhoWaitForThisObject: null
c's threadsWhoWaitForThisObject: pa
~~~~

在第一次pb执行期间，三个对象的**threadWhoEnterTheMonitor**以及**threadsWhoWaitForThisObject**的变化如下：
~~~~
a's threadWhoEnterTheMonitor: pb -> null
b's threadWhoEnterTheMonitor: pb -> null
c's threadWhoEnterTheMonitor: null

a's threadsWhoWaitForThisObject: pb
b's threadsWhoWaitForThisObject: null
c's threadsWhoWaitForThisObject: pa
~~~~

在第一次pc执行期间，三个对象的**threadWhoEnterTheMonitor**以及**threadsWhoWaitForThisObject**的变化如下：
~~~~
a's threadWhoEnterTheMonitor: null
b's threadWhoEnterTheMonitor: pc -> null
c's threadWhoEnterTheMonitor: pc -> null

a's threadsWhoWaitForThisObject: pb
b's threadsWhoWaitForThisObject: pc
c's threadsWhoWaitForThisObject: pa -> null
~~~~

在第一次执行pa和pb的期间，由于prev对象对应的**threadsWhoWaitForThisObject**是null，所以实际上**self.notify()**是不会起作用的．

而在第一次执行pc的期间，**c's threadsWhoWaitForThisObject**开始是pa，所以是会唤醒pa的．等到pc的synchronized执行完后，此时尽管三个对象的**threadWhoEnterTheMonitor**都是null，但是此时pb和pc都没有被唤醒，所以不存在竞争的问题．

后面的迭代就跟第一次基本上差不多了，各位可以自行尝试走一遍．
