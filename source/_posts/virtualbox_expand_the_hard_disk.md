---
title: virtualbox扩大硬盘
date: 2016-02-7 21:32:29
tags: virtualbox
toc: false
---
前段时间由于误删了vm虚拟机，痛苦很久。于是就把家里的vb虚拟机拷贝过来用。
分享两个vb的小知识：
**1、增加硬盘大小**
   很多人都会想到增加一块磁盘，公司有同事培训也是这么提议的。
   这个方法肯定能解决虚拟机磁盘过小的问题。网上也有详细步骤。
   这个方法的让有强迫症的童鞋很不爽，比如我，在工作目录下挂载一个磁盘，总觉得不  爽。
我的做法是扩大原来的磁盘。就好比把你碗里的饭倒到一个大盆子里面。
用到的工具：
vb自带的VBoxManage 
一张Ubuntu安装盘，官网下载的包就可以了
Gparted 工具 
a、调整大小
命令使用如下：
![](http://img.blog.csdn.net/20140813101855235?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanlwZWl0YW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
我把我的磁盘调整为500G
D:\Program Files (x86)\Oracle\VirtualBox>
VBoxManage modifyhd E:\Ubuntu\Ubuntu.vdi --resize 512000
b、关闭虚拟机，关闭插入Ubuntu安装盘，开机。意义就是从光盘启动，然后就能随意操作磁盘。这和U盘里装一个操作系统是一个作用。选择try ubuntu 启动系统。
 ![](http://img.blog.csdn.net/20140813101921739?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanlwZWl0YW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](http://img.blog.csdn.net/20140813101826593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanlwZWl0YW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
c、系统中安装gpartted ，这里可以选择其他工具哈。亲测的是gparted，易上手。
![](http://img.blog.csdn.net/20140813102042485?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanlwZWl0YW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 
选择更改大小和移动哈，不要去分区神马的。上面的图不是实际图片哈。通过光盘启动之后，上面的设备是没被锁定的，可以调整大小和移动。总之很简单，谁用谁知道。
然后，然后就没有然后了，取出光盘，开机吧！
 
 
**2、开机自动启动vb**
   新建一个StartUbuntu.bat ，名字随意哈，就一句话。
   把它放到win7 的Start Menu\Programs\Startup
   "D:\Program Files (x86)\Oracle\VirtualBox\VBoxManage" startvm Ubuntu64 --type gui
 
   Ubuntu64 是我的虚拟机名字。不知道自己的就用命令查看VBoxManage list vms

