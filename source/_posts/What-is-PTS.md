---
title: PTS测试
date: 2018-1-25 20:37:01
tags: [Bluetooth,PTS]
---

# 什么是PTS
全称Profile Tuning Suite，[官网](https://www.bluetooth.com/develop-with-bluetooth/test-tools/profile-tuning-suite)说得很明白。一套自动测试HCI以上部分是否满足规范的程序。也是为了保证蓝牙的一致性。


# PTS测试的意义
意义官网也有说，是站在SIG的角度来说的。下面说说我的理解。
- 对产品来说
 - PTS测试是一次蓝牙交互兼容性检查，早发现问题，早解决
- 对开发同学来说
 - 可以进一步学习蓝牙规范上的知识，加深理解
 - 如果实验室测试出问题，也可以坐到心中有底，能快速复现


# 获取PTS配置文件
如果自己开发一个蓝牙设备，清楚每个细节，自己设置PTS的配置文件是极好的。
如果用别人的过了认证的方案，那就直接拿来用就好。
1. 在线走一遍[Launchstudio](#%20Launchstudio)认证流程，自动生成PTS配置文件，利用这个文件来过进行PTS测试。
当然直接找实验室要这个文件也是不错的
好处是：
- 生成的配置文件和实验室一致
- 如果遇到有实验室测试失败的情况，本地可以快速验证
![](/imgs/What-is-PTS/1516798258529.png)

2. 下载别人认证好的PTS配置文件，比如QCOM或者MTK
好处是：
- 简单
- 可以把厂家声称支持的全部跑一遍，有问题请他们解决
![](/imgs/What-is-PTS/1516798132762.png)

# PTS测试硬件
需要一个dongle，[Bluetoothstore](https://bluetoothstore.org/)有售。
整个一个store就卖这一个产品，有趣。

![](/imgs/What-is-PTS/1516872041427.png)

# PTS测试
剩下就是下载软件，然后开始测试。按照提示操作即可，如果测试失败就debug吧。
![Alt text](/imgs/What-is-PTS/1516879438197.png)



# Launchstudio
QCOM或者MTK他们的蓝牙方案都是过认证了的，可以在launchstudio查到。

为了过认证，实验室都会根据目前的蓝牙方案，给出一个合理的认证方式。
[Launchstudio](https://launchstudio.bluetooth.com/)是2017年才上线的一个工具，现在可以直接在线List Products，实验室会帮忙建立项目，我们也可以自己建立。

走实验室，一般都是选择Required Testing。在我们的手机上，目前较多的方案是host subsystem + controller subsystem，我认为直接选择No Required Testing也是可以的，反正最后都是花大价钱买Declaration ID，再加上确实不会做什么修改。按照实验室某专家的说法，不Testing有风险，我觉得可能是实验室也要有生意。和实验室合作，有问题他们会帮忙处理。
![](/imgs/What-is-PTS//1516778495642.png)





