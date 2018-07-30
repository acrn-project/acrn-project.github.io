---
layout: post
title:  "提高DM接收线程优先级提升iperf TCP性能"
categories: ACRN
tags: ACRN
author: Jie Deng
description: 
---

#### 背景

Improve proirity of DM mevent thread (that is current virtio-net BE RX thread). For high traffic, the TX thread occupies a large amount of CPU, which influences the receipt of the TCP ACK packet in RX thread. As a result, it restricts the iperf TCP performance to go up. 

1. 保持UOS上iperf处于运行状态

2. 在SOS上运行htop工具 ([htop download](/assets/res/htop.tar))
   ```
   > export LD_LIBRARY_PATH=LD_LIBRARY_PATH:./
   > ./htop
   ```

   ![htop](/assets/images/htop.png)

3. 选择mevnet接收线程，按F7将其NI值调到-20, 观察TCP实时速度（注意， 你的系统上mevent线程pid不一定是332， 一般如果UOS iperf在运行, 运行htop后CPU占用最高的就是，具体可以通过cat /proc/<pid>/status 查看其Name来确认）
