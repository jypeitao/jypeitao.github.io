---
title: Android 模拟器
date: 2016-03-10 15:33:56
tags: Tools
---


自从做手机项目以后就再没用过Android的模拟器啦。一个是有许多定制，模拟器不一定能跑起来。最重要的是，项目有相应的真机用来开发。

Android模拟器对APP开发还是很有用的。

Official Android Emulator在我的Ubuntu系统上跑不起来，CPU不支持KVM。之前在Windows下用过，不过感觉有点卡。

Genymotion是一款著名的Android模拟器。

# Genymotion介绍

Genymotion是由一家叫[genymobile](http://www.genymobile.com)的公司开发。
![](/imgs/genymotion-for-android/photo-equpe.jpg)


Genymotion号称最快的模拟器，实际上说它是一个虚拟机更合适。可以在[Genymotion](http://www.genymobile.com/genymotion/)页面看到官方的介绍。
> 
Make better Apps with the fastest Emulator in the world

# Ubuntu14.04下安装Genymotion
## 1. Genymotion账号注册
[点击这里创建账号](https://www.genymotion.com/account/create/)


 
## 2. Virtual box 安装
> 
Genymotion operation relies on the use of Oracle VM VirtualBox in the background. This enables virtualizing Android operating systems. If you do not already have Oracle VM VirtualBox installed, you will be asked to do so prior to installing Genymotion.
**If you already have Oracle VM VirtualBox, note that we recommend version 5.0.4 or above.**

官方推荐版本是5.04或者更高。
可以通过apt-get的方式安装，但是用Ubuntu14.04默认的source安装的版本比较老。
选择直接下载最新版本virtualbox-5.0_5.0.16-105871-Ubuntu-trusty_amd64.deb
[官方下载地址](https://www.virtualbox.org/wiki/Linux_Downloads)

下载下来之后，直接双击通过软件中心安装。
![](/imgs/genymotion-for-android/vb_install.png)

运行起来之后如下图
![](/imgs/genymotion-for-android/vb_run.png)


## 3. Genymotion安装

进入[Genymotion download ](https://www.genymotion.com/download/) 下载对应的版本。
我下载的是genymotion-2.6.0-linux_x64.bin

sudo chmod +x genymotion-2.6.0-linux_x64.bin && ./genymotion-2.6.0-linux_x64.bin

命令行下运行：
./genymotion

![](/imgs/genymotion-for-android/genymotion.png)

genymotion界面很简介，各个按钮功能也很清楚。
首先用之前注册的账号登陆一下，然后点击Add按钮，会列出很多可用的virtual devices，选择之后开始下载。

![](/imgs/genymotion-for-android/genymotion_download_vd.png)

# Android studio的Genymotion Plugin安装

[官方](https://docs.genymotion.com/Content/04_Tools/Genymotion_Plugin_for_Android_Studio/Installing_the_plugin.htm)提供了两种方式
- JetBrains repository method (recommended)
- Manual method

手动方式我没找到下载的地方！囧

我用的是推荐方法：
1. 启动 Android Studio
2. 进入 File/Settings
3. 选择 Plugins 并点击 Browse repositories
4. 输入genymotion 就会找到
5. 点击安装



