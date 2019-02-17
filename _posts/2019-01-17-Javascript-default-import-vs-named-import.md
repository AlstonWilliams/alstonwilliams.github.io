---
layout: post
title: Javascript-default-import-vs-named-import
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Javascript
tags:
- Javascript
---
在写React Native代码时,遇到了一个问题.

在一个组件中用**import {React, Component} from 'react';**导入**React**依赖时,老是提示**undefined is not a function(evaluating '_react.React.createElement')**.

由于调试起来特别不方便,我们又不能从错误栈中得到什么有用的信息,所以搞得我有点懵.捣鼓了好长时间没解决.

后来,发现**React Native**中自带的**index.android.js**中,导入**React**依赖用的是下面这条语句:**import React, {Component} from 'react';**

就想碰碰运气,修改了一下,结果竟然真成了!!!!

你们能想象当时我心中一万个草泥马奔腾而过的场景吗?!

于是,就查了查这两者到底有什么区别.

下面是从Stack Overflow上面得到的答案.

感觉**default import**和**named import**翻译成中文有点难翻译,索性就不翻译了.直接用英文昵称得了.

## default import

下面这种形式就是**default import**:
**import A from './A'**

只有**A.js**中,包含了一条**export default **语句时,它才能生效.比如,**export default 42**.

不管你是从**A.js**中import什么,比如:
**import A from './A'
import MyA from './A'
import Something from './A'**

它们对应的都是解析成导入**A.js**中的**export default**对应的对象.

## named import

**named import**是下面这种形式:
**import {A} from './A'**

只有当**A.js**中包含了一条export A的语句时,它才会生效.比如:
**export const A = 42**

在**named import**,括号中的名字很关键!

如果**A.js**中只有**export const A = 42**这条语句,那下面的语句都将不会生效:

**
import {MyA} from './A';
import {Something} from './A';
**

为了让上面的那两条语句生效,你需要在**A.js**中添加如下内容:

**
export const MyA = 42;
export const Something = 42;
**

## 结合
一个**js**文件中,只能有一个**default export**语句,但是可以有多个**named export **.例如,如果你的**A.js**内容如下:
**export default 42
export const myA = 43
export const Something = 44**

那你可以通过下面的语句将它们导入进来:
**import A, {myA, Something} from './A';**

## 链接
[StackOverflow上的解释](http://stackoverflow.com/questions/36795819/when-should-i-use-curly-braces-for-es6-import)
