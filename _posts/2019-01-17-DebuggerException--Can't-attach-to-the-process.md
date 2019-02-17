---
layout: post
title: DebuggerException--Can't-attach-to-the-process
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
修改**/etc/sysctl.d/10-ptrace.conf**，将**kernel.yama.ptrace_scope = 1**修改成**kernel.yama.ptrace_scope = 0**
