---
layout: post
title:  "VBS-K Framework Virtualization Overhead Analysis"
categories: ACRN
tags: ACRN
author: Jie Deng
description: 
---

# Introduction 
<br>

In this post, we will evaluate the VBS-K framework overhead through a testing virtual device called virtio-echo. The total overhead of a frontend-backend application based on VBS-K contains of `VBS-K framework overhead` and `application specific overhead`. The application specific overhead depends on its specific frontend-backend design, from microseconds to seconds. While for a fixed hardware platform, the average overhead of VBS-K framework is fixed, in our HW case, the overall VBS-K framework overhead is on the microsecond level which can meet the needs of most applications.

<br>

# Architecture of VIRTIO-ECHO
<br>

virtio-echo is a virtual device based on virtio designed for testing of ACRN virtio backend service in kernel (VBS-K) framework. It includes a virtio-echo frontend driver, a virtio-echo driver in ACRN device model (DM) for initialization and a virtio-echo driver based on VBS-K for data reception and transmission. For more virtualization background introduction, please refer to below ACRN hypervisor introduction and Virtio high level design.

> [Introduction to Project ACRN](https://projectacrn.github.io/latest/introduction/index.html) <br>
[Virtio high-level design in ACRN](https://projectacrn.github.io/latest/developer-guides/virtio-hld.html)

virtio-echo is implemented as a virtio legacy device in the ACRN device model (DM), and which is registered as a PCI virtio device to the guest OS (UOS). The software of virtio-echo consists of three parts:

- **virtio-echo Frontend Driver:**
  This drvier runs in the UOS. It prepares the RXQ and notify the backend for receiving incoming data when the UOS starts. Second, it copys the received data from the RXQ to TXQ and sends it to backend. After receiving the message that the transmission is completed, it starts again a new round of reception and transmission, and keep running until a specified number of cycle is reached.

- **virtio-echo Drvier in DM:**
  This driver is used for configration in the initialization. It simulates a virtual PCI device for frontend driver use and sets necessary information, such as device configuration, virtqueue information to the VBS-K. After initialization, all data exchange will be taken over by VBK-K vbs-echo drvier.

- **vbs-echo Backend Driver:**
  This driver sets all frontend RX buffer to be a specific value and send the data to the frontend driver. The frontend driver receives the data in RXQ and copys it to the TXQ and send it back to the backend. The backend driver then notify the frontend driver that the data in the TXQ is sucessfully received. Actually, the backend driver doesn't prcoess or use the received data.

Figure 1 shows the whole architecture of virtio-echo.

<br>

![virtio-echo_architecture](/assets/images/acrn-vbsk/virtio-echo_architecture.png)
<p align="center">Figure 1: virtio-echo Architecture</p>

<br>

# Virtualization Overhead Analysis

<br>

What is the overhead of VBS-K framework? 

As we know, the VBS-K is designed to handle notification in SOS kernel instead of in SOS user space DM. This can avoid the overhead due to switching between kernel space and user space. Virtqueues are allocated by UOS and the virtio-echo driver in DM configures the virtqueue information to VBS-K backend so that virtqueues can be shared between UOS and SOS. So there is no copy overhead in this sense. The overhead of VBS-K framework mainly contains two parts, i.e., `kick overhead` and `notify overhead`.

- **Kick Overhead:**
  The UOS gets trapped when it executes sensitive instruction and hypervisor will first get notified. The notification is then assembled into IOREQ and saved in a shared IO page and forwarded to VHM module by hypervisor. The VHM notifies its client for this IOREQ, in this case, the client is vbs-echo backend driver. `kick overhead` is defined as the interval from the beginning of UOS trap to specific VBS-K drvier e.g. virtio-echo gets notified.

- **Notify Overhead:**
  After the data in virtqueue processed by backend driver, vbs-echo calls the VHM module to inject interrupt into the frontend. The VHM then uses the hypercall provided by hyperviosr which causes the UOS VMEXIT. The hypervisor finally inject interrupt into vLAPIC of UOS and resume it. The UOS therefore receives an interrupt notification. `notify overhead` is defined as the interval from the beginning of interrupt injection to UOS starts interrupt processing.

The overhead of a specific application based on VBS-K includes two parts, i.e., VBS-K framework overhead and application specific overhead.

- **VBS-K Framework Overhead:**
  As defined above, VBS-K framework overhead refers to kick overhead and notify overhead.

- **Application Specific Overhead:**
  A specific virtual device has its own frontend driver and backend driver. The application specific overhead depends on its own design.

<br>

Figure 2a shows the overhead of one end to end operation in virtio-echo. Overhead of steps marked as red are caused by virtualization scheme based on VBS-K framework. Costs of one “*kick*” operation and one “*notify*” operation are both on microsecond level. Overhead of steps marked as yellow depend on specific frontend and backend driver of virtual devices. For virtio-echo, the whole end to end process from step1 to step9 costs about dozens of microsecond. That's because virtio-echo does little things in its frontend and backend driver which is just for testing and there is little process overhead.

<br>

![virtio-echo_overhead1](/assets/images/acrn-vbsk/virtio-echo_overhead1-notime.png)
<p align="center">Figure 2a: End to End Overhead of virtio-echo</p>

<br>

Figure 2b details the path of kick and notify operation showed in Figure 2a. The VBS-K framework overhead is caused by operations through these paths. As we can see, all these operations are processed in kernel mode which avoids extra overhead of passing IOREQ to user space processing.

<br>

![virtio-echo_overhead2](/assets/images/acrn-vbsk/virtio-echo_overhead2.png)
<p align="center">Figure 2b: Path of VBS-K Framework Overhead</p>

<br>

# Conclusion

<br>

Different from VBS-U processing in user mode, VBS-K move things into the kernel mode and can be used to accelerate the processing. A virtual device virtio-echo based on VBS-K framework is used to evaluate the VBS-K framework overhead. In our test, the VBS-K framework Overhead (one kick operation and one notify operation) is on the microsecond level which can meet the needs of most applications.

<br>