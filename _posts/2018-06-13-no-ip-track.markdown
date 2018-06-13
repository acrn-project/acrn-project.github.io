---
layout: post
title:  "BKM for NO IP"
categories: ACRN
tags: ACRN
author: Jie Deng
description: BKM for checking the no ip problem
---

检查公司网络策略
============

公司有些网口只允许一个IP, 这会导致UOS拿不到IP的情况. 建议使用路由器连接MRB板子。或者使用双网卡PC, 其中一个网卡连公司，另一个绑定DHCP服务器并将其与MRB连接，这样该PC网卡和MRB上的所有OS都由PC上的DHCP分配IP.

检查是否UOS死机
============

当使用自动脚本启动时并不进入UOS控制台，如果此时发现ping不通UOS，应首先排查UOS是否死机. 如果没有死机，则可通过minicom从SOS进入UOS，具体方法如下:

1. 如果SOS没有minicom则需安装
2. minicom -D /dev/pts/0 (/dev/pts/0 will change according to your use case)
3. run ifconfig check the network status

检查是否由于不同UOS版本
=============

1. 如果仅仅是两个UOS版本不同，而一个有IP，一个没有IP，这是由于UOS本身造成的.
2. 比较是否由于不同UOS启动脚本所产生的不同配置错误造成的.

ifconfig手动配置一个IP看能否工作
=============

如果手动给UOS配置一个合理IP后（ifconfig ethX ip address）数据可以通，则不是virtio-net的原因，DHCP过程不是由virtio-net控制，请检查上述原因.


分析需提供的日志
=============

如果通过上述检查任然不能确定，请提供如下材料
1. sos: ifconfig 打印输出
2. uos: ifconfig 打印输出
3. uos: 启动日志
4. wireshark抓包文件 (在SOS启动之前，PC上安装运行wireshark抓包工具, 抓取同意网段下所有数据包, 用以检查dhcp包)