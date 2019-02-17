---
layout: post
title: Ubuntu-17-10安装TensorFlow
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Tensorflow
tags:
- Tensorflow
---
## 环境

- OS: Ubuntu 17.10
- 有无GPU: 无
- 处理器位数: 64位
- Python版本: python3.6
- pip版本: pip9.0.1

## 安装过程

照着官网上的步骤来走，中间遇到了一些问题。所以这里也做了相应的修改。

1. 安装必要的包
~~~
$ sudo apt-get install python3-pip python3-dev python-virtualenv
~~~

2. 创建目录
~~~
$ mkdir ~/tensorflow
$ cd ~/tensorflow
$ virtualenv --system-site-packages -p python3 venv 
~~~

3. 进入到virtualenv环境中
~~~
$ source ~/tensorflow/venv/bin/activate
~~~

4. 安装tensorflow包
官网上说的是，执行`pip install -U tensorflow`这条命令就好了，但是我这儿一直安装不成功。就换了一种方法。

~~~
$ python3.6 -m pip install https://storage.googleapis.com/tensorflow/linux/cpu/tensorflow-1.10.1-cp36-cp34m-linux_x86_64.whl
~~~

由于我的机器上，`python`命令默认的版本是`3.7`，而tensorflow没有给python 3.7的包，所以会安装失败，于是就直接用`python3.6`这条命令。

另外，上面的url选择的时候，要选择正确，比如`linux`代表你的机器的类型，如果写错了，比如写成`mac`，那么即使安装成功，导入Tensorflow包时，也会报错。`cpu`表示的是使用**cpu**，如果你有**gpu**的话，可以写成`gpu`.另外如果你的python版本不是3.6，那么`cp36`也要对应的修改一下。

当然，由于墙的问题，上面的命令，你也很可能不会执行成功。有两个方案：
- 1. 给Terminal加socks5或者HTTP(S)代理。
- 2. 手动去上述站点上下载下来这个文件，然后安装。当然，这个操作的前提是，你的浏览器能翻墙。

我这边是采用的第二种方案，下载下来后，通过下面的命令进行安装:

~~~
$ python3.6 -m pip install file:///home/alstonwilliams/Downloads/tensorflow-1.10.1-cp36-cp36m-linux_x86_64.whl
~~~

5. 尝试有没有安装成功

打开`python3.6`，输入
~~~
# Python
import tensorflow as tf
hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
~~~

如果输出为
~~~
Hello, TensorFlow!
~~~

那么一切OK，可以开始编写Tensorflow程序了。
