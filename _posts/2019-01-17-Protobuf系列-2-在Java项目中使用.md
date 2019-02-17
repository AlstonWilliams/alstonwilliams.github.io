---
layout: post
title: Protobuf系列-2-在Java项目中使用
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Protocol-Buffer
tags:
- Protocol-Buffer
---
在上一篇文章中，我们介绍了如何安装Protoc，以及编译一个**proto**文件．

在这篇文章中，我们将会介绍如何在项目中使用这些编译出来的文件．

## 创建一个maven项目

通过下面这条命令创建一个最简单的maven项目就好:
**mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false**

它会在当前目录下生成一个名为**my-app**的项目．

这个项目的初始结构如下：

![](http://upload-images.jianshu.io/upload_images/4108852-7ed9686d937eb50d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到，其中生成的的**App.java以及AppTest.java**，我们并不需要，所以直接给删掉就好了．

然后，我们需要向其中添加一个**protobuf**的依赖，因为生成的Java文件需要这个依赖中的内容．

在**pom.xml**中添加如下内容：

~~~~
    <dependency>
      <groupId>com.google.protobuf</groupId>
      <artifactId>protobuf-java</artifactId>
      <version>3.4.0</version>
    </dependency>
~~~~

注意上面的那个版本应该跟你安装的protoc版本一致．

## 创建proto文件并生成Java文件

我们在项目的根目录下，创建一个**src/main/java/proto**文件夹，并编写一个名为**addressbook.proto**的文件:

~~~~
syntax = "proto3";

package com.mycompany.app;

option java_package = "com.mycompany.app";
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
~~~~

注意上面的这个文件中，**option java_package**部分，应该给你的项目中你想存放生成的文件的包名一致．

通过**protoc --java_out=. addressbook.proto**命令，可以生成Java源文件．

![](http://upload-images.jianshu.io/upload_images/4108852-08a4b879a9e34e79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后，我们需要将生成的这些源文件拷贝到项目根目录下的**src/main/java/com/mycompany/app**目录中．

![](http://upload-images.jianshu.io/upload_images/4108852-631da217d69b21dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里需要注意的是，如果你在写**proto**文件时，其中**option java_package**选项指定的目录跟你想存放的目录不一样的话，需要手动修改一下生成的文件中的包名．

然后，我们在**src/main/java/com/mycompany/app**下，创建一个Java源文件，其内容如下：

~~~~
package com.mycompany.app;

import java.io.*;

public class AddPerson {

    static Person promptForAddress(BufferedReader stdin, PrintStream stdout) throws IOException {

        Person.Builder person = Person.newBuilder();

        stdout.print("Enter person ID: ");
        person.setId(Integer.valueOf(stdin.readLine()));

        stdout.print("Enter name: ");
        person.setName(stdin.readLine());

        stdout.print("Enter email address (blank for none): ");
        String email = stdin.readLine();
        if (email.length() > 0)
            person.setEmail(email);

        while (true) {

            stdout.print("Enter a phone number (or leave blank to finish)");
            String telephone = stdin.readLine();
            if (telephone.length() == 0)
                break;

            Person.PhoneNumber.Builder phoneNumber = Person.PhoneNumber.newBuilder();
            phoneNumber.setNumber(telephone);

            stdout.print("Is this a mobile, home, or work phone? ");
            String type = stdin.readLine();
            if (type.equals("mobile"))
                phoneNumber.setType(Person.PhoneType.MOBILE);
            else if (type.equals("home"))
                phoneNumber.setType(Person.PhoneType.HOME);
            else if (type.equals("work"))
                phoneNumber.setType(Person.PhoneType.WORK);
            else {
                stdout.print("Unknown phone type. Using default");
                phoneNumber.setType(Person.PhoneType.HOME);
            }

            person.addPhones(phoneNumber);

        }

        return person.build();

    }

    public static void main(String[] args) throws Exception {

        if (args.length != 1) {
            System.err.println("Usage: AddPerson ADDRESS_BOOK_FILE");
            System.exit(-1);
        }

        AddressBook.Builder addressBook = AddressBook.newBuilder();

        try {
            addressBook.mergeFrom(new FileInputStream(args[0]));
        } catch (FileNotFoundException e) {
            System.out.println(args[0] + " doesn't exist. Creating file.");
        }

        addressBook.addPeople(promptForAddress(new BufferedReader(new InputStreamReader(System.in)), System.out));

        FileOutputStream fileOutputStream = new FileOutputStream(args[0]);
        addressBook.build().writeTo(fileOutputStream);
        fileOutputStream.close();

    }

}

~~~~

它的作用是，让你输入一个Person的信息，并且保存到你指定的文件中．

那么如何编译这个文件呢？

进入到**src/main/java**这个目录下，执行下面的命令：

** javac -cp .:/opt/mvn/resp/com/google/protobuf/protobuf-java/3.4.0/protobuf-java-3.4.0.jar com/mycompany/app/AddPerson.java**

其中你需要把**protbuf-java**的路径换成你的机器上的路径．

编译完成后，通过**java -cp .:/opt/mvn/resp/com/google/protobuf/protobuf-java/3.4.0/protobuf-java-3.4.0.jar com/mycompany/app/AddPerson　hello**这条命令来运行刚刚**AddPerson**文件．输入完Person的信息后，我们会看到在当前目录下，会生成一个十六进制的hello文件．

我们打开这个16进制文件，可以看到其中的内容为：

~~~~
0A 08 0A 04 66 73 64 61 10 01
~~~~

其中保存了一些元数据以及我输入的数据的十六进制编码．

就这样就完成了．

## 总结

在这篇文章中，我们一步步介绍了如何创建一个Java并和protobuf结合．

这篇文章中，我只介绍了一部分用法，很不全面．

请去查看[官方文档](https://developers.google.com/protocol-buffers/docs/javatutorial)来获取更加全面的信息．

关于Protobuf的官方文档，后续我可能会翻译出来，也是放在这个系列中．
