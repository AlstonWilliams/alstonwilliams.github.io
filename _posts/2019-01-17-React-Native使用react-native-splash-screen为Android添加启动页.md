---
layout: post
title: React-Native使用react-native-splash-screen为Android添加启动页
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Javascript
tags:
- Javascript
---
现在我们做一个APP,不可避免的要添加一个启动页,使APP对用户更加友好.废话少说,我们现在就来介绍如何利用react-native-splash-screen为Android APP实现启动页.

需要注意的是,如果你是为IOS实现,那么步骤是不一样的.具体请看官网.

## 步骤

### 添加依赖
在项目根目录下执行下面这条命令:**npm i react-native-splash-screen --save**
然后,我们还需要链接一下:**react-native link**

### 创建布局文件
在**android/app/src/main/res**目录中,新建**layout**目录,在此目录下,创建**launch_screen.xml**文件,其内容如下:
~~~
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/launch_screen">
</LinearLayout>
~~~

### 添加启动页图片
在**android/app/src/main/res**目录中,新建**drawable-xhdpi和drawable-xxhdpi**目录,在其中放入启动页图片,命名为**launch_screen.png**

### 修改Android源文件
修改**android/app/src/main/java/com/meizhuang/MainActivity.java**文件,添加如下内容:
~~~
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        SplashScreen.show(this);
        super.onCreate(savedInstanceState);
    }
~~~

修改完成后,如下图所示:

![](http://upload-images.jianshu.io/upload_images/4108852-0427d1bd5a56e541.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 修改项目的js文件
这里就仁者见仁智者见智了, 具体修改哪个文件,要看你想放在哪了.我这里因为就一个js文件,**index.android.js**,所以是直接放在了这个文件中.在项目的根组件中,添加如下内容:
~~~
import SplashScreen from 'react-native-splash-screen'

export default class WelcomePage extends Component {

    componentDidMount() {
    	 // do anything while splash screen keeps, use await to wait for an async task.
        SplashScreen.hide();
    }
}
~~~

我这里稍微修改了一下,让其暂停两秒后,再跳转到主页面:

![](http://upload-images.jianshu.io/upload_images/4108852-449f5ec07d7bc5ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后重新安装打包并安装APP到手机上,就会看到启动页
