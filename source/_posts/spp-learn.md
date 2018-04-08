---
title: Serial Port Profile 学习
date: 2016-03-08 11:04:40
tags: Bluetooth
---

# 蓝牙SPP
SPP(Serial Port Profile)定义了用蓝牙替代RS232串口的相关协议和交互过程。

下图展示了蓝牙的profile的架构和各个profiles之间的依赖关系。
![Bluetooth Profile Structure](/imgs/spp-learn/Bluetooth_Profile_Structure.png)

一个profile依赖另一个profile ，图上表示出来就是间接或者直接包含。

比如：
**Serial Port Profile 依赖 Generic Access Profile**
**File Transfer Profile 依赖 Generic Object Exchange Profile -> Serial Port Profile**

# SPP概述
![Profile Stack](/imgs/spp-learn/Profile_Stack.png)
Baseband,LMP和L2CAP对应OSI参考模型的物理层和链路层。RFCOMM规定了传输协议。SDP就是蓝牙服务发现协议。
SPP中一个设备发起连接，另一个设备等待被连接。一次只能有一个连接。
当一个连接被发起，发现服务的过程就会被执行。
Master 和Slave的角色也是可转换的。

# 应用层
SPP是建立在GAP之上的，因此，GAP中的所以强制要求部分对SPP也是强制的。
**创建链接和虚拟串口连接**
1.   提交一个服务发现请求，找到远程设备的RFCOMM服务通道号(RFCOMM Server channel number)。当然，如果知道已经知   道了要连接的服务，这个请求可以省略。
2.  进行远程设备认证和加密，这个过程的可选择的。
3.  为远程RFCOMM实体申请一个新的L2CAP通道。
4.  在L2CAP的通道上开启一个FRCOMM回话。
5.  在RFCOMM会话上启动一个新的数据链路连接

经过上面5步，虚拟串口连接就准备好了。

**注意:**如果在启动数据链路连接之前RFCOMM会话已经存在，新的连接必须建立在已有的RFCOMM会话之上，换句话说就是直接省略了步骤3和步骤4。
步骤1和步骤2的顺序也是没要求的。


**接受连接和建立虚拟串口连接**
1. 根据远程设备的请求，参与认证或者加密过程
2. 接受从L2CAP发起的新的通道连接
3. 接受从RFCOMM发起的新的通道连接
4. 接受从RFCOMM会话发起的新的数据连接

**在本地SDP数据库注册服务信息**
可以通过RFCOMM的所有服务或应用都需要提供SDP服务信息。

# L2CAP的互操作性要求

SPP只用面向连接的L2CAP通道。这也意味着广播数据和单行的无连接的数据不会被SPP用到。

对L2CAP MTU(Maximum Transmission Unit)大小SPP不强加任何限制在
  
 服务质量的协商是可选项。

建议L2CAP的错误控制特性被用来配置SPP的重传模式

# SDP的互操作性要求
SPP的发起者是没有SDP服务记录的。
在SPP v1.2之前，BluetoothProfileDescriptorList属性不是强制的，因此即使是后面的版本也要假设没有这个属性。
下图就是SDP服务记录的结构。
![SDP Service Record](/imgs/spp-learn/SDP_Service_Record.png)

# 链路控制和管理交互性需求
这个部分其实参考Link Manager Protocol 就好。

下面是Link Controller的状态图(Core_v4.2)
![State diagram of link controller](/imgs/spp-learn/State_diagram_of_link_controller.png)

**inquiry scan：** 设备处于可被发现状态
**inquiry：**扫描状态 ，扫描周边设备
**Page：** 连接状态
**Page Scan：** 等待被连接状态
