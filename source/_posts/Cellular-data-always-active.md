---
title: Cellular data always active
date: 2016-08-04 14:22:30
tags: [Android]
---

# Cellular data always active
## 作用：保持数据连接处于激活状态，以便实现快速切换。
打开wifi，打开数据连接，如果没有勾选Cellular data always active，数据流量的网络接口是不存在的。
反之，就会看到数据流量的网络接口。
![](/imgs/Cellular-data-always-active/aa.jpg)
wlan0就是wifi连上AP的结果，rmnet_data0就是数据连接。



## 实现

首先从开发者选项中的开关看起。

![](/imgs/Cellular-data-always-active/SequenceDiagram1.jpg)


 这里假设用户勾选了该选项 enable 值为true。
```java
private void handleMobileDataAlwaysOn() {
        final boolean enable = (Settings.Global.getInt(
                mContext.getContentResolver(), Settings.Global.MOBILE_DATA_ALWAYS_ON, 0) == 1);
        final boolean isEnabled = (mNetworkRequests.get(mDefaultMobileDataRequest) != null);
        if (enable == isEnabled) {
            return;  // Nothing to do.
        }
        if (enable) {
            handleRegisterNetworkRequest(new NetworkRequestInfo(
                  null, mDefaultMobileDataRequest, new Binder(), NetworkRequestInfo.REQUEST));
        } else {
            handleReleaseNetworkRequest(mDefaultMobileDataRequest, Process.SYSTEM_UID);
        }
    }
```
mDefaultMobileDataRequest是在ConnectivityService 实例化的时候创建。
从注释中可以看出它的用途，keep mobile data active。
```java
// Request used to optionally keep mobile data active even when higher
    // priority networks like Wi-Fi are active.
    private final NetworkRequest mDefaultMobileDataRequest;
```
NetworkRequest 是个什么类？
```java
/**
 * Defines a request for a network, made through {@link NetworkRequest.Builder} and used
 * to request a network via {@link ConnectivityManager#requestNetwork} or listen for changes
 * via {@link ConnectivityManager#registerNetworkCallback}.
 */
public class NetworkRequest implements Parcelable {
```

![](/imgs/Cellular-data-always-active/diagram.png)


下面看一下handleRegisterNetworkRequest的实现。
```java
private void handleRegisterNetworkRequest(NetworkRequestInfo nri) {
        mNetworkRequests.put(nri.request, nri);
        mNetworkRequestInfoLogs.log("REGISTER " + nri);
        if (!nri.isRequest) {
            for (NetworkAgentInfo network : mNetworkAgentInfos.values()) {
                if (nri.request.networkCapabilities.hasSignalStrength() &&
                        network.satisfiesImmutableCapabilitiesOf(nri.request)) {
                    updateSignalStrengthThresholds(network, "REGISTER", nri.request);
                }
            }
        }
        rematchAllNetworksAndRequests(null, 0);
        if (nri.isRequest && mNetworkForRequestId.get(nri.request.requestId) == null) {
            sendUpdatedScoreToFactories(nri.request, 0);
        }
    }

```
实现中的nri.request 就是上面的mDefaultMobileDataRequest。 可以看到mDefaultMobileDataRequest被放入mNetworkRequests中。nri.isRequest就是NetworkRequestInfo.REQUEST ，值是true。
接着就会执行一个很关键的函数，rematchAllNetworksAndRequests。
```java
 /**
     * Attempt to rematch all Networks with NetworkRequests.  This may result in Networks
     * being disconnected.
     * @param changed If only one Network's score or capabilities have been modified since the 
     *       last time this function was called, pass this Network in this argument, otherwise 
     *       pass null.
     * @param oldScore If only one Network has been changed but its NetworkCapabilities have 
     *       not changed, pass in the Network's score (from getCurrentScore()) prior to the 
     *       change via this argument, otherwise pass {@code changed.getCurrentScore()} or 
     *       0 if {@code changed} is {@code null}. This is because NetworkCapabilities 
     *       influence a network's score.
     */
    private void rematchAllNetworksAndRequests(NetworkAgentInfo changed, int oldScore) {
    if (changed != null && oldScore < changed.getCurrentScore()) {
            rematchNetworkAndRequests(changed, ReapUnvalidatedNetworks.REAP);
        } else {
            final NetworkAgentInfo[] nais = mNetworkAgentInfos.values().toArray(
                    new NetworkAgentInfo[mNetworkAgentInfos.size()]);
            // Rematch higher scoring networks first to prevent requests first matching a lower
            // scoring network and then a higher scoring network, which could produce multiple
            // callbacks and inadvertently unlinger networks.
            Arrays.sort(nais);
            for (NetworkAgentInfo nai : nais) {
                rematchNetworkAndRequests(nai,
                        // Only reap the last time through the loop.  Reaping before all 
                        // rematching is complete could incorrectly teardown a network 
                        // that hasn't yet been rematched.
                        (nai != nais[nais.length-1]) ? ReapUnvalidatedNetworks.DONT_REAP
                                : ReapUnvalidatedNetworks.REAP);
            }
        }
    }
```
值得注意的是，这里的nais是一个score降序排列。第一次看到这里肯定想知道mNetworkAgentInfos是从哪里来的。


![](/imgs/Cellular-data-always-active/SequenceDiagram2.jpg)

``` java
/**
 * A Utility class for handling for communicating between bearer-specific
 * code and ConnectivityService.
 *
 * A bearer may have more than one NetworkAgent if it can simultaneously
 * support separate networks (IMS / Internet / MMS Apns on cellular, or
 * perhaps connections with different SSID or P2P for Wi-Fi).
 *
 * @hide
 */
public abstract class NetworkAgent extends Handler {
```
NetworkAgent是一个abstract类，需要具体子类。它继承Handler，本质上就是一个Handler。

在下面这些文件中有NetworkAgent的子类：
 Vpn.java
 DataConnection.java
 WifiStateMachine.java
 EthernetNetworkFactory.java
 BluetoothTetheringNetworkFactory.java

这里先不去跟踪具体的NetworkAgent初始化流程了。回头看rematchAllNetworksAndRequests。
因为这里假设的情况是，先打开数据连接，然后打开wifi连接一个热点，mNetworkAgentInfos中只有一个元素。下面就是一个实际的例子

![](/imgs/Cellular-data-always-active/networkinfo.jpg)

接下来一步是sendUpdatedScoreToFactories。

![](/imgs/Cellular-data-always-active/NetworkFactorys.jpg)

高通项目实现了QtiDctController ,所以在看代码的时候，有些函数是在QtiDctController.java 文件中。
``` java
public class QtiDctController extends DctController {
```

sendUpdatedScoreToFactories中利用AsyncChannel通知各个NetWorkFactories。

![](/imgs/Cellular-data-always-active/reques_network.jpg)

经过上面的流程，modem端就做好了准备， 状态机DataConnection的状态会InactiveState ->ActivatingState->ActiveState.

在进入ActiveState的时候，会实例化DcNetworkAgent，**DcNetworkAgent的父类就是NetworkAgent**，然后执行**registerNetworkAgent**函数。

下面看看ConnectivityService.java 中的实现
``` java
public int registerNetworkAgent(Messenger messenger, NetworkInfo networkInfo,
            LinkProperties linkProperties, NetworkCapabilities networkCapabilities,
            int currentScore, NetworkMisc networkMisc) {
        enforceConnectivityInternalPermission();

        // Instead of passing mDefaultRequest, provide an API to determine whether a Network
        // satisfies mDefaultRequest.
        final NetworkAgentInfo nai = new NetworkAgentInfo(messenger, new AsyncChannel(),
                new Network(reserveNetId()), new NetworkInfo(networkInfo), new LinkProperties(
                linkProperties), new NetworkCapabilities(networkCapabilities), currentScore,
                mContext, mTrackerHandler, new NetworkMisc(networkMisc), mDefaultRequest, this);
        synchronized (this) {
            nai.networkMonitor.systemReady = mSystemReady;
        }
        addValidationLogs(nai.networkMonitor.getValidationLogs(), nai.network);
        if (DBG) log("registerNetworkAgent " + nai);
        mHandler.sendMessage(mHandler.obtainMessage(EVENT_REGISTER_NETWORK_AGENT, nai));
        return nai.network.netId;
    }
```
实例化NetworkAgentInfo ，封装了Messenger，NetworInfo，currentScore等数据。
EVENT_REGISTER_NETWORK_AGENT的处理还是在ConnectivityService.java 中，handleRegisterNetworkAgent。
``` java
private void handleRegisterNetworkAgent(NetworkAgentInfo na) {
        if (VDBG) log("Got NetworkAgent Messenger");
        mNetworkAgentInfos.put(na.messenger, na);
        synchronized (mNetworkForNetId) {
            mNetworkForNetId.put(na.network.netId, na);
        }
        na.asyncChannel.connect(mContext, mTrackerHandler, na.messenger);
        NetworkInfo networkInfo = na.networkInfo;
        na.networkInfo = null;
        updateNetworkInfo(na, networkInfo);
    }
```
多线程，mTrackerHandler是在另一个线程中处理，na.asyncChannel.connect会发一个CMD_CHANNEL_HALF_CONNECTED的消息，mTrackerHandler中处理。同时函数updateNetworkInfo被另外线程调用。

直接看下重要的函数**updateNetworkInfo**的部分代码。
``` java
private void updateNetworkInfo(NetworkAgentInfo networkAgent, NetworkInfo newInfo) {
       ...
        if (state == NetworkInfo.State.CONNECTED && !networkAgent.created) {
            try {
                // This should never fail.  Specifying an already in use NetID will cause failure.
                if (networkAgent.isVPN()) {
                    mNetd.createVirtualNetwork(
                      networkAgent.network.netId,
                      !networkAgent.linkProperties.getDnsServers().isEmpty(),
                      (networkAgent.networkMisc == null ||
                       !networkAgent.networkMisc.allowBypass)
                    );
                } else {
                    mNetd.createPhysicalNetwork(
                      networkAgent.network.netId,
                      networkAgent.networkCapabilities.hasCapability(
                        NET_CAPABILITY_NOT_RESTRICTED) ?
                      null : NetworkManagementService.PERMISSION_SYSTEM
                    );
                }
            } catch (Exception e) {
                loge("Error creating network " + networkAgent.network.netId + ": "
                     + e.getMessage());
                return;
            }
            networkAgent.created = true;
            updateLinkProperties(networkAgent, null);
            notifyIfacesChanged();
            ...
        } else if (state == NetworkInfo.State.DISCONNECTED) {
            networkAgent.asyncChannel.disconnect();
            if (networkAgent.isVPN()) {
                synchronized (mProxyLock) {
                    if (mDefaultProxyDisabled) {
                        mDefaultProxyDisabled = false;
                        if (mGlobalProxy == null && mDefaultProxy != null) {
                            sendProxyBroadcast(mDefaultProxy);
                        }
                    }
                }
            }
        } 
    }
```
重要的是通过Netd去创建物理网络，更新连接属性。
至此，实现流程基本就看完了。


## 总结
打开wifi 连上AP 并且打开数据连接，勾选Cellular data always active，此时数据连接依然是激活的。

当断开wifi的时候，也会走到**updateNetworkInfo**，能够迅速切换到数据连接。因为和RIL沟通，连接状态的改变等耗时的操作已经完成了。只需要把wifi连接断开就可以了。

关于断开wifi ，切换到数据流量流程请参考：
http://blog.csdn.net/u010961631/article/details/48629601


# Aggressive WLAN to Cellular handover
## 作用：系统会在WLAN信号较弱时，主动将网络模式从WLAN网络切换到移动网络
实际上，WIFI和数据流量的自动切换始于L版本，系统总是使用得分高的网络。
英文版本的描述更准确些：
>When enabled,WLAN will be more aggressive in handing over the data connection to Cellular, when WLAN signal is low

## 实现
还是先从开发者选项看起。
![](/imgs/Cellular-data-always-active/aggressiveHandover.jpg)

调用了一个API，透过binder，改变了WifiStateMachine中的一个变量值。仅此而已。
被用到的地方有一处，在下面这个函数中
``` java
 private void calculateWifiScore(WifiLinkLayerStats stats) {
```

![](/imgs/Cellular-data-always-active/1469518020474.png)

可以看到，这里故意减少RSSI (received signal strength indication)，使得[网络评分](http://10.120.10.100:9002/Network_score)的时候得分较小，进而切换到数据网络。

calculateWifiScore的调用地方也只有一处。

当状态机处于L2ConnectedState状态的时候，简单说来就是wifi已经连上网了
```java
/* Connected at 802.11 (L2) level */
private State mL2ConnectedState = new L2ConnectedState();
```

``` java

case CMD_RSSI_POLL:
     if (message.arg1 == mRssiPollToken) {
         if (mWifiConfigStore.enableChipWakeUpWhenAssociated.get()) {
             WifiLinkLayerStats stats = getWifiLinkLayerStats(VDBG);
             if (stats != null) {
                 if (mWifiInfo.getRssi() != WifiInfo.INVALID_RSSI
                                        && (stats.rssi_mgmt == 0
                                        || stats.beacon_rx == 0)) {
                     stats = null;
                 }
             }
             fetchRssiLinkSpeedAndFrequencyNative();
             calculateWifiScore(stats);
         }
         sendMessageDelayed(obtainMessage(CMD_RSSI_POLL,
                            mRssiPollToken, 0), POLL_RSSI_INTERVAL_MSECS);
     } else {
         // Polling has completed
     }
     break;
```
从上面的代码可以看出来，这里是个循环，不断的发送消息CMD_RSSI_POLL，不断的calculateWifiScore。除非状态机的状态发生了变化。

计算分数干啥？

calculateWifiScore根据wifi情况，动态修改分数，wifi的信号越弱，分数越低。

之前说过，这个feature就是故意减少wifi信号强度值。 假设本来强度还可以，得分能到50分，结果故意减少信号强度值，最后得分就只有40了。同时假设数据流量网络得分是45分。因为40比45分小，故切换网络。

## 总结

这个功能有啥用处？
可能就是一个优化方向吧。除非WLAN状况非常好，否则还是用移动数据好了。

Cellular data always active也有尽量用移动数据的意思在里面。

这其中更深层次的意义我还没理解到。

参考链接：
http://android.stackexchange.com/questions/90250/what-does-the-aggressive-wi-fi-to-cellular-handover-option-in-developer-settin
http://blog.csdn.net/u010961631/article/details/48970107


# Always allow WLAN Roam Scans

## 作用：一律运行WLAN漫游扫描

## 实现
这个实现和Aggressive WLAN to Cellular handover极为相似，最终影响的变量是alwaysEnableScansWhileAssociated。

被用到的地方也只有一处，也是在L2ConnectedState状态中。

``` java

case CMD_START_SCAN:
      ... 
      if (mWifiInfo.txSuccessRate > mWifiConfigStore.maxTxPacketForPartialScans
              || mWifiInfo.rxSuccessRate > mWifiConfigStore.maxRxPacketForPartialScans) {
          // Don't scan if lots of packets are being sent
          restrictChannelList = true;
          if (mWifiConfigStore.alwaysEnableScansWhileAssociated.get() == 0) {
              messageHandlingStatus = MESSAGE_HANDLING_STATUS_REFUSED;
              return HANDLED;
          }
      }
      ...
      // scan
```
当有大量数据传输的时候，如果在执行扫描，性能肯定有影响，所以直接返回了。
>Don't scan if lots of packets are being sent


## 总结
这个feature有啥作用？ 没什么用吧！ 

https://android.googlesource.com/platform/frameworks/opt/net/wifi/+/c6f06c628ee3583b60ff31a7da442e0ac7b26d97%5E%21/service/java/com/android/server/wifi/WifiStateMachine.java
>autojoin tuning, making LTE handover less aggressive
>Bug: 15700122
>Change-Id: I2836abf791f8c71c63a901170f247c4030e8e8f9
``` java
+    private int mAggressiveHandover = 0;
+
+    int getAggressiveHandover() {
+        return mAggressiveHandover;
+    }
+
+    void enableAggressiveHandover(int enabled) {
+        mAggressiveHandover = enabled;
+    }
+
+    private int mAllowScansWithTraffic = 1;
+
+    public void setAllowScansWithTraffic(int enabled) {
+        mAllowScansWithTraffic = enabled;
+    }
+
+    public int getAllowScansWithTraffic() {
+        return mAllowScansWithTraffic;
+    }
+
```

从提交来看，Always allow WLAN Roam Scans和 Always allow WLAN Roam Scans都是这次提交添加的。从commit log来看是在tuning autojoin 功能。


参考链接：
https://supportforums.cisco.com/discussion/12543176/workaround-android-devices-do-not-roam-wifi
