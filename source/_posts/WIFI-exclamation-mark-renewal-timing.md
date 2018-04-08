---
title: WIFI exclamation mark renewal timing
date: 2016-03-24 13:24:17
tags: [WIFI, Android]
---
# 0x00 前言
----
从Android L开始，当wifi连接之后，系统会尝试去连接google服务器，如果访问成功就表示wifi能上网。
如果不能成功访问就会在wifi旁边显示一个感叹号。

 > 不同版本默认的地址不一样
**Android 6.0**  http://connectivitycheck.gstatic.com/generate_204
**Andorid 5.1**  http://connectivitycheck.android.com/generate_204
**Android 5.0**  http://clients3.google.com/generate_204

关于这个功能的说明，网上也能找到一大堆啦。

本文着重写下感叹号更新的流程和时机。

# 0x01 关键代码说明
----
函数`int isCaptivePortal()`是用来判断是否能够连接google服务器的函数，连接成功返回204。

该功能可以通过adb shell来关闭。
>**adb shell settings put global captive_portal_detection_enabled 0**

默认是google的服务器，由于GFW的原因不能成功连接。可以修改这个地址。
>**adb shell settings put global captive_portal_server www.qualcomm.cn **

修改后的地址对应的服务器一定要有**generate_204**，否则会被认为连接失败。

遇到连接失败，可以先**adb shell settings get global captive_portal_server** 查看下是否有值。
如果有值，再确认下是否有**generate_204**。

函数`public NetworkCapabilities[] getDefaultNetworkCapabilitiesForUser(int userId)`也很重要，用来获取所有的NetworkCapabilities对象，这些对象记录了是否连接成功。


# 0x02 按进程分析该问题
----
状态栏图标的显示是由systemui负责的，而是否连接成功由system_server判断。

状态栏负责显示，不管是连接成功还是失败，总得有人去通知它更新界面。

最容易想到的方式是在通知状态栏更新图标的时候把需要目前的状态一起传递给systemui。

实际实现是system_server通知需要更新图标状态，由systemui主动获取获取连接情况，再更加连接情况来决定是否显示感叹号。




 **systemui**
 systemui通过监听广播事件`ConnectivityManager.CONNECTIVITY_ACTION`和`ConnectivityManager.INET_CONDITION_ACTION`来得到图标更新请求。


之前也提到，更新成什么图标，是根据连接状态来的，连接状态是通过`getDefaultNetworkCapabilitiesForUser`从ConnectivityService拿到的NetworkCapabilities对象中获取的。

![](/imgs/WIFI-exclamation-mark-renewal-timing/updateicon.jpg)

注意上图最右边的`getCurrentIconId`和`inetConditon`
```java
public int getCurrentIconId() {
        if (mCurrentState.connected) {
            return getIcons().mSbIcons[mCurrentState.inetCondition][mCurrentState.level];
        } else if (mCurrentState.enabled) {
            return getIcons().mSbDiscState;
        } else {
            return getIcons().mSbNullState;
        }
    }
```

 **system_server**
ConnectivityService类发送广播通知systemui更新状态。

NetworkMonitor类中的`isCaptivePortal`是很关键的一个函数。故从这里开始讲起。

每一个NetworkAgentInfo实例持有一个NetworkMonitor对象。NetworkMonitor本质是一个状态机。
关于NetworkAgentInfo，代码中有如下注释：
 


> 
 A bag class used by ConnectivityService for holding a collection of most recent
 information published by a particular NetworkAgent as well as the
 AsyncChannel/messenger for reaching that NetworkAgent and lists of NetworkRequests
 interested in using it.  Default sort order is descending by score.
 
NetworkAgentInfo和NetworkMonitor本文不做详细介绍。

只要知道，在wifi连接之后，状态机切换到`EvaluatingState`，在这个状态调用`isCaptivePortal`。
如果`isCaptivePortal`返回204就表示连接成功，进入新状态`ValidatedState`

这个状态很简单，贴一下代码。
```java
private class ValidatedState extends State {
        @Override
        public void enter() {
            mConnectivityServiceHandler.sendMessage(obtainMessage(EVENT_NETWORK_TESTED,
                    NETWORK_TEST_RESULT_VALID, 0, mNetworkAgentInfo));
        }
        @Override
        public boolean processMessage(Message message) {
            switch (message.what) {
                case CMD_NETWORK_CONNECTED:
                    transitionTo(mValidatedState);
                    return HANDLED;
                default:
                    return NOT_HANDLED;
            }
        }
    }
```

![](/imgs/WIFI-exclamation-mark-renewal-timing/NET_CAPABILITY_VALIDATED.jpg)


# 0x03 行为不正常的分析点
如果出现了图标的显示和预期不一致，检测点如下：

1. `NetworkControllerImpl.java`的`void onReceive(Context context, Intent intent)`有没有收到广播`ConnectivityManager.CONNECTIVITY_ACTION`或者`ConnectivityManager.INET_CONDITION_ACTION`。

2. `NetworkControllerImpl.java`的`updateConnectivity`有没有符合期望的`mValidatedTransports`。

3. `NetworkMonitor`的`isCaptivePortal`是否有被调用。


