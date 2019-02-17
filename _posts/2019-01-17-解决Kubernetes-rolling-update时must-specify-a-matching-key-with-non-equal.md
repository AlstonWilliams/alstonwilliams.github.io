---
layout: post
title: 解决Kubernetes-rolling-update时must-specify-a-matching-key-with-non-equal
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Kubernetes
tags:
- Kubernetes
---
本来rolling-update时，新创建一个rc文件，并更新其中的内容，然后使用**kubectl rolling-update rc_name -f rc_file**就可以更新成功了．但是，实际操作过程中，遇到了一个问题，就是如题所示的错误．具体的原因，请参考底部链接的文章．

解决方案很简单，就是使用**kubectl rolling-update rc_name --image=new_image_path**．这样命令反而更加简单．

参考链接:[Kubernetes 滚动升级| 褚哥说|](http://valleylord.github.io/post/201603-kubernetes-roll/)
