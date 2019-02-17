---
layout: post
title: Protobuf系列-3-gRPC-Protobuf插件安装
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Protocol-Buffer
tags:
- Protocol-Buffer
---
研究Protocol Buffer的目的，就是因为项目中需要用到gRPC．所以在搞明白Protocol Buffer之后，由于gRPC在Protocol Buffer的语法中增加了一些新的内容，所以我们还需要为Protobuf编译一个gRPC插件．

## 环境

由于gRPC插件跟Protobuf密切相关，所以，我们需要先明确相关的环境．

- Protobuf: 3.4.0
- gRPC: 1.8.x

## Protobuf

尽管我们可以直接从Protobuf 的github仓库中找到可用的release版本，但是由于gRPC的Protobuf插件在编译时，还需要引用一些文件，而这往往在可用的release版本中是不可用的．

所以，我们需要先编译Protobuf.

依次执行下面的命令：

~~~~

sudo apt-get install autoconf automake libtool curl make g++ unzip

cd /tmp

git clone https://github.com/google/protobuf.git && cd protobuf

git checkout v3.4.0

./autogen.sh

./configure --prefix=/opt/share/protobuf/ppc64le/

make && make install

export PATH=/opt/share/protobuf/ppc64le/bin/:$PATH

export LD_LIBRARY_PATH=/opt/share/protobuf/ppc64le/lib/:$LD_LIBRARY_PATH

export CXXFLAGS="-I/opt/share/protobuf/ppc64le/include -L/opt/share/protobuf/ppc64le/lib/"                                              
export LDFLAGS="-L/opt/share/protobuf/ppc64le/lib"
export PROTOC=/opt/share/protobuf/ppc64le/bin/protoc  

~~~~

上面的命令就是将**protoc**安装到**/opt/share/protobuf/ppc64le**并设置了一些编译gRPC插件需要的环境变量．

## 编译gRPC插件

在上面的环境变量设置好之后，依次执行下面的命令即可进行编译：

~~~~
cd tmp

git clone https://github.com/grpc/grpc-java.git && cd grpc-java

git checkout v1.8.x

cd compiler

../gradlew java_pluginExecutable
~~~~

如果一切顺利，则会出现**BUILD SUCCESSFUL**．

然后，我们可以通过下面的命令测试是否构建成功．

**../gradlew test**

![](http://upload-images.jianshu.io/upload_images/4108852-54bcb59ab5f79d9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果结果如上图所示，那么就表示没有问题．

如果构建过程中遇到了问题，那么请移步[这里](https://github.com/grpc/grpc-java/issues/2487)来查看有没有解决方案．

实际上，我就是在捣鼓了一天之后，才搞成功的．

## 使用gRPC Protobuf插件

在构建完成之后，我们要如何来使用它呢？

使用下面这条命令即可：

**protoc --plugin=protoc-gen-grpc-java=build/exe/java_plugin/protoc-gen-grpc-java \
  --grpc-java_out="$OUTPUT_FILE" --proto_path="$DIR_OF_PROTO_FILE" "$PROTO_FILE"**

当然，你需要将上面的命令中的gRPC插件的位置等替换一下．

比如，在我的机器上，我写了这么一个proto：

~~~~
syntax = "proto3";

package com.project;

option java_package = "com.projecthome";
option java_multiple_files = true;
option java_outer_classname = "AddressBookProtos";

message Person {

    string name = 1;
    int32 id = 2;
    string email = 3;

    enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }

    message PhoneNumber {

        string number = 1;
        PhoneType type = 2;

    }

    repeated PhoneNumber phones = 4;

}

message AddressBook {

    repeated Person people = 1;

}

service AddressBookOperation {
	rpc getAddressBook(AddressBook) returns(AddressBook){}
}

~~~~

在上面这个文件的底部，有一个**service**的定义，如果用纯protoc来编译的话，是不会成功的，因为它是gRPC中的语法．所以我们需要通过下面的命令进行编译：

**protoc --plugin=protoc-gen-grpc-java=/home/alstonwilliams/source_code/grpc-java/compiler/build/exe/java_plugin/protoc-gen-grpc-java --grpc-java_out=. --proto_path=. addressbook.proto**

即把这个proto编译到本地．编译完成之后，我们就能看到对应的Java代码：

![](http://upload-images.jianshu.io/upload_images/4108852-49dc028f8a744161.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

个人感觉其实编译gRPC的Protobuf插件还是蛮困难的，一是因为版本问题，二是官方文档中的描述并不是很清晰，有好多环境变量都没有详细的说到．

希望这篇文章能够顺利地帮助你编译成功．
