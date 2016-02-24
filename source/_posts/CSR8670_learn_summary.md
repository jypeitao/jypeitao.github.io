---
title: CSR8670_learn_summary
date: 2016-02-24 09:25:00
tags: Bluetooth
---
#CSR8670介绍

[CSR8670™](http://www.csr.com/cn/products/csr8670) 是一款音频片上系统 (SoC) 解决方案，配有无线连接功能、嵌入式闪存和集成式触控传感器，使功能丰富的家庭娱乐系统和可穿戴音频产品能够带来无与伦比的用户体验和卓越的音频性能。

CSR8670有一个[XAP processor](https://en.wikipedia.org/wiki/XAP_processor) 和一个[Kalimba DSP](http://www.icinnovate.com/)。

Kalimba  DSP一般被用来处理音频数据,能被Developer写代码操作寄存器。
>在BlueCore3-Mulitmedia的基带控制器(MCU)中运行蓝牙协议栈以及Handfree Profile和>Handset　Profile，并从蓝牙的同步面向连接(SCO)链路中提取语音信息并转送给Kalimba >DSP，Kalimba DSP中的软件完成语音处理，再经过数模转换到扬声器放出来，反方向是模>数转换从麦克风中收集语音信息，到Kalimba DSP中进行处理，然后传送给基带控制器，再>通过蓝牙的SCO链路发送出去。


**ADK**：Audio Development Kit

**CSR “Firmware” **：蓝牙协议栈代码和一些硬件的控制代码，直接用来操作硬件。这些代码跑在**XAP processor**上面。

**DSP Application**：跑在**Kalimba DSP**上的代码。

**VM Application**：也是跑在**XAP Processor**上的代码，但是跑在**CSR Firmware**的一个**Sandbox area**，对硬件的访问能力有限。有个特点就是只有在Firmware不忙的时候才运行，实时性较差。
 
![processor architecture](/imgs/processor_architecture.bmp)



#ROM的组成
![rom](/imgs/rom.bmp)
ROM包括Filessystem 和 CSR Firmware两大块。Filessystem 由VM Application、DSP Application和其它一些数据文件组成。
蓝牙协议栈和硬件控制代码构成Firmware。

#VM Application示例
![architecture of the application](/imgs/architecture_of_the_application.bmp)

VM Application由c语言实现，调用现成的蓝牙的库函数，由于提供了库函数源码，理论上也是可以去修改的。通过tasks 和messages去调度程序执行。

**Tasks**是用于构建应用程序基本组成单元,为提供firmware提供了一个回调接口。
**Messages**是Tasks 之间用来传递消息的。由Task t, MessageId id, Message payload组成。
- Task t:  用来标示消息的目标
- MessageId id：区分不同消息
- Message payload：额外信息
```cpp
#include <pio.h>
#include <stdio.h>
#include <message.h>

#define BUTTON_A        (1 << 0)        /* PIO0 is BUTTON_A */
#define BUTTON_B        (1 << 1)        /* PIO1 is BUTTON_B */
#define BUTTON_C        (1 << 2)        /* PIO2 is BUTTON_C */
#define BUTTON_D        (1 << 3)        /* PIO3 is BUTTON_D */

typedef struct
{
    TaskData    task;   /* task is required for messages to be delivered */
} appState;

appState app;

static void app_handler(Task task, MessageId id, Message message) 
{
    switch (id)
    {
    case MESSAGE_PIO_CHANGED:
        handle_pio(task, (MessagePioChanged*)message);
        break;

    default:
        printf("Unhandled message 0x%x\n", id);
    }
}

static void handle_pio(Task task, MessagePioChanged *pio)
{
    if (pio->state & BUTTON_A) printf("Button A pressed\n");
    if (pio->state & BUTTON_B) printf("Button B pressed\n");    
}

int main(void)
{
    /* Set app_handler() function to handle app's messages */
    app.task.handler = app_handler;

    /* Set app task to receive PIO messages */
    MessagePioTask(&app.task);

    /* Setup PIO interrupt messages */
    PioDebounce32(BUTTON_A | BUTTON_B,  /* PIO pins we are interested in */
                2, 20);                 /* 2 reads and 20ms between them */

    MessageLoop();
    
    return 0;
}

/* End-of-File */

```
