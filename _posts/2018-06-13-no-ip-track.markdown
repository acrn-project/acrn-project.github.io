---
layout: post
title:  "SOS/UOS NO IP的一些case"
categories: Misc
tags: Misc
author: Jie Deng
description: BKM for checking the SOS/UOS no ip problem
---

## 检查公司网络策略
============

公司可能会对switch连多个设备进行检查，导致拿不到IP的情况. 建议使用家用路由器建立一个内网供MRB板子使用.如果是连接公司端口,碰到没有IP或很长时间才有IP的情况,请第一时间联系IT同事 Wei, JiachenX (11650945) jiachenx.wei@intel.com, 报告端口号和设备MAC地址，检查相关情况.

## 检查网络中是否有"干扰DHCP服务器"
============

公司不允许在内网私自搭建DHCP服务器. 下图展示了一个"干扰DHCP服务器 192.168.10.1"的例子(公司的DHCP服务器是10开头的). 导致最终获取的IP不是10.x.x.x造成网络不通.如遇此种情况，请联系IT.

![rogue_dhcp_server](/assets/images/rogue_dhcp.png)

## 检查switch设备
============

为了方便板子上外网，实验室普遍采用一个switch连接到公司, 并在其上连接多个设备, 测试之前除了考虑是否有公司网络策略影响外，还应确保用的swtich设备的每个端口都是好的.

## 检查是否UOS死机
============

当使用自动脚本启动时并不进入UOS控制台，如果此时发现ping不通UOS，应首先排查UOS是否死机. 如果没有死机，则可通过minicom从SOS进入UOS检查，具体方法如下:

1. 如果SOS没有minicom则需安装
2. minicom -D /dev/pts/0 (/dev/pts/0 will change according to your use case)
3. run ifconfig check the network status

## 检查UOS网络配置文件
============

如果UOS文件系统中有如下文件应该删除. 

```
/usr/lib/systemd/network/50-acrn.netdev 
/usr/lib/systemd/network/50-acrn.network
/usr/lib/systemd/network/50-acrn_tap0.netdev 
/usr/lib/systemd/network/50-eth.network 
```

在给UOS配置双虚拟网卡时，以上文件会导致如下loop网络，最终被IT发现违反公司网络策略，物理端口被封.

![loop_network](/assets/images/loop_net.jpg)

## 检查是否由于不同UOS版本
=============

1. 如果仅仅是两个UOS版本不同，一个版本有IP，另一个版本没有IP或DHCP很慢，这是由于UOS本身OS造成的.
2. 比较是否由于不同UOS启动脚本所产生的不同配置错误造成的.

## ifconfig手动配置一个IP看能否工作
=============

如果手动给UOS配置一个合理IP后(ifconfig ethX ip address, 尽量不要连接公司路由，因为公司路由可能连有很多机器，手动配置可能会有重复IP问题 .建议使用双网卡的模式或家用路由的连接方法), 数据可以通，则不是virtio-net的原因，DHCP过程不是由virtio-net控制，请检查上述原因.


## 分析需提供的日志
=============

如遇问题通过上述情况检查任然不能确定，请提供如下材料

1. 出问题的版本号(commit id号)和之前好的版本号(commit id号)
2. sos: ifconfig or ip a 打印输出
3. uos: ifconfig or ip a 打印输出
4. uos: 启动日志
5. wireshark抓包文件
  
   - 在SOS启动之前，PC上安装运行wireshark抓包工具, 抓取同一网段下所有数据包.
   - 配置switch端口镜像, 以便MRB板子连接的端口收发的包能转发到PC连接的端口上，具体设置方法参见具体的switch说明书([一个例子](https://www.tp-link.com/us/faq-527.html)).
   - 设置PC网卡 ip link set ethX promisc on, 这样PC上运行的wireshark软件才可以抓到所有数据包

   ![DCHP Capture Setup](/assets/images/dhcp_capture.png)

6. DHCP日志，具体方法如下
  * Add below line into the file /usr/lib/systemd/system/systemd-networkd.service
     ```
     [Service]
     ... 
     Environment=SYSTEMD_LOG_LEVEL=debug
     ```
  * Sync and Reset MRB
  * journalctl -b -u systemd-networkd > networkd.log

7. 运行 sudo dhclient -v 输出的打印   
