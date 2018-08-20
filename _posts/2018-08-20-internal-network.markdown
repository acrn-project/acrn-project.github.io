---
layout: post
title:  "SOS/UOS Communication without External Network "
categories: Misc
tags: Misc
author: Jie Deng
description: 
---

# Introduction 
<br>

In some cases, you just want to communicate between SOS and UOSes without external netowrk, even SOS does not have a physical network card at all.
How can we achieve this.

<br>
You may need to create something like following network infrastructure
<br>

![architecture](/assets/images/acrn-interal-net/architecture.png)

<br>

If you launch SOS without connecting network cable, you can never get IP from a external DHCP server.

<br>

![no_net_sos](/assets/images/acrn-interal-net/no_net_sos.png)

<br>

Configure a static IP to the bridge in SOS. E.g.
> ifconfig acrn-br0 192.168.1.1

<br>

![configure_sos_ip](/assets/images/acrn-interal-net/configure_sos_ip.png)

<br>

Similarly, you will launch the UOS without IP

<br>

![no_net_uos](/assets/images/acrn-interal-net/no_net_uos.png)

<br>

Configure a static IP to the NIC in UOS and then you can ping the SOS from UOS. E.g.
> ifconfig enp0s4 192.168.1.2 <br> ping 192.168.1.1

<br>

![ping_uos_to_sos](/assets/images/acrn-interal-net/ping_uos_to_sos.png)
