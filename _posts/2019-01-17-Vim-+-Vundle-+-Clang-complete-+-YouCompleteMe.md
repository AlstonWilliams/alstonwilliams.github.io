---
layout: post
title: Vim-+-Vundle-+-Clang-complete-+-YouCompleteMe
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 小工具
tags:
- 小工具
---
在Vim中,用C语言写demo,虽然加上NERDTree插件之后,写起来很方便,但是,我们仍然需要手动输入很多代码,比如变量名等.而在现在的IDE中,却有自动提示功能,我们只需要输入首字母,然后选择对应的变量名或者方法名,就能节省很多的输入时间.

在Vim中,手动输入很多demo的完整的代码后,终于忍不了了.找了两个插件来完成自动提示的功能.

下面会详细介绍插件的安装过程.

## Vundle
我们首先介绍Vundle,Vundle是Vim的插件管理工具,使用它,我们可以很方便的管理插件,而不用**git clone**克隆插件到*~/.vim/bundle*目录下,手动进行安装了.现在,我们只需要动动手指头,修改一下*~/.vimrc*文件中的内容,就能安装插件啦.

那么如何安装它呢?这个插件的安装过程,相对还是比较简单的,只需要下面一条命令:
**git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim**

安装完成后,我们还需要配置一下Vundle.在*~/.vimrc*这个文件中的开头处,插入以下内容:


![](http://upload-images.jianshu.io/upload_images/4108852-0e491c0692a79337.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个文件的内容,可以在Vundle的github上的页面中看到,但是,需要注意的是,我们需要删除我们并不需要的内容.请自行查看官方文档.

有了Vundle之后,Clang-complete和YouCompleteMe的安装就简单的多啦.

## Clang-complete
Clang-complete是一个为c/c++而生的代码自动完成的插件.当我们输入*.*和*->*后,会给我们提示.

那我们如何安装它呢?

我们需要先通过下面的命令安装其依赖的工具以及库:
**sudo apt-get install libclang-dev clang**

然后,通过Vundle安装它.在**~/.vimrc**文件中,在**call vundle#begin()**和**call vundle#end()**之间,添加这行**Plugin 'rip-rip/clang_complete'**.添加完之后,如下图所示:


![](http://upload-images.jianshu.io/upload_images/4108852-4c54e6d31afb072f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样,我们需要配置一下这个插件.还是在**~/.vimrc**文件中,添加其需要的库的位置.在此文件的最后,加上这一行:
**let g:clang_library_path='/usr/lib/llvm-3.4/lib'**

需要注意的是**g:clang_library_path**这个变量的值,要是你的机器上的安装路径,因为版本的原因,很可能和我这里的路径不同.你需要替换成你的路径,一般来说,和上面的路径相比,只是版本号不同.

然后,打开Vim,输入**: PluginInstall**,就会自动安装**~/.vimrc**这个文件中配置的插件:


![](http://upload-images.jianshu.io/upload_images/4108852-fd93bfb0319ed4bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中的最左边的那一列,是你输入上面的安装命令之后,才会出来的.

这样,就安装成功了.你可以写一个c文件尝试一下:


![](http://upload-images.jianshu.io/upload_images/4108852-2fa734d073046e11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## YouCompleteMe

这个插件和**Clang-complete**插件一样,都是用于自动提示的插件.那我们为啥需要**YouCompleteMe**呢?因为**Clang-complete**的功能很有限,只有我们输入**.**或者**->**之后,它才会给我们自动提示,而我们还想要有自动提示变量名,方法名的功能.于是,就采用了**YouCompleteMe**这个插件.

而这个插件的安装,也是最为复杂的.

首先,因为要求Vim 版本在7.3.584及以上,而Ubuntu的源中的vim版本已经较老了,我们需要重新安装一个新版本的Vim.

我们可以选择编译源代码来安装,也可以选择从其他的ppa来安装.这里我们选择比较简单的方式,从ppa安装.

先安装需要的依赖:
**sudo apt-get install libncurses5-dev libgnome2-dev libgnomeui-dev libgtk2.0-dev libatk1.0-dev libbonoboui2-dev libcairo2-dev libx11-dev libxpm-dev libxt-dev python-dev ruby-dev mercurial cmake**

然后使用下面的命令来安装新版本的Vim:
**sudo add-apt-repository ppa:fcwu-tw/ppa
sudo apt-get update
sudo apt-get install vim**

安装完后,我们可以通过下面的命令来检查Vim的版本:
**dpkg -s vim | grep 'Version'**

我安装完后,是**7.4.273**版本.

然后,通过Vundle安装它.在**~/.vimrc**文件中,在**call vundle#begin()**和**call vundle#end()**之间,添加这行**Plugin 'Valloric/YouCompleteMe'**.

添加完后,如下图所示:

![](http://upload-images.jianshu.io/upload_images/4108852-0ea4489438bb3827.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样,通过在Vim中输入**: PluginInstall**命令来安装此插件.

这个插件的安装,可能会比较慢,请耐心等待.

然后,我们需要执行下面的这两条命令,使其真正可用:
**cd ~/.vim/bundle/YouCompleteMe
./install.py --clang-completer**

我们写一个C文件来尝试一下这个插件:
![](http://upload-images.jianshu.io/upload_images/4108852-413f32f17c2bb7f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结
这里介绍的都是基础的安装,并没有涉及到了解并使用这些插件.请自行查看文档来了解.
