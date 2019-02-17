---
layout: post
title: ApacheBench发送KeepAlive请求，收到的响应却是Connnection-Close
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
服务器环境是Tomcat.

如果你仔细观察ApacheBench发送的请求，会发现它发送的都是HTTP 1.0协议的请求．而Tomcat似乎仅对HTTP 1.1 提供Keep-Alive.对于收到的HTTP 1.0 的请求，不管是否制定了Keep-Alive,都会返回Connection:Close
