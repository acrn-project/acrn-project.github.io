---
layout: post
title:  "Network Virtualization in ACRN"
categories: ACRN
tags: ACRN
author: Jie Deng
description: 
---

# Introduction 
<br>

The virtio-net is the paravirtualization solution used in ACRN. The ACRN device model emulate virtual NICs for UOS use and the frontend virtio network driver operate the virtual NIC. Both of them follow the virtio specification. If you are not familiar with the virtualization, I highly recommend you reading following articles before you going on.

> [Introduction to Project ACRN](https://projectacrn.github.io/latest/introduction/index.html) <br>
[Virtio high-level design in ACRN](https://projectacrn.github.io/latest/developer-guides/virtio-hld.html)


<br>

# Supported Features
<br>
-	Legacy device supported, modern device not supported
-	Two virtqueues are used in virtio-net, i.e. RX queue and TX queue
-	Virtio indirect descriptor supported
-	TAP backend supported
-	Control queue not supported
-	NIC multiple queue not supported

<br>

# Architecture of Network Virtualization
<br>

Sending the network data from UOS to the outside requires lots of modules working together. It involves UOS network stack, virtio-net frontend driver, ACRN hypervisor, VHM module in SOS, ACRN device model, virtio-net backend driver, SOS network stack, bridge, tap device, IGB driver and so on. Figure 1 shows the network virtualization architecture in ACRN. This figure links many necessary network virtualization components together for easy understanding. The green components are parts of ACRN solution while the gray components are parts of Linux kernel.

<br>

![network-virtualization-architecture](/assets/images/acrn-vnet/network-virtualization-architecture.png)
<p align="center">Figure 1: Network Virtualization Architecture</p>

<br>

- **SOS/UOS Network Stack:**
  This is the standard Linux TCP/IP stack which is today's most feature-rich TCP/IP implementation.

- **virtio-net Frontend Driver:**
  This is the standard driver in Linux Kernel for virtual Ethernet device. This driver matches devices with PCI vendor ID 0x1AF4 and PCI Device ID 0x1000 (for legacy devices in our case) or 0x1041 (for modern devices). The virtual NIC supports two virtqueues, one for transmitting packets and the other for receiving packets. The frontend driver places empty buffers into one virtqueue for receiving packets and enqueues outgoing packets into another virtqueue for transmission. The size of each virtqueue is 1024 which is configurable in virtio-net backend driver. 

- **ACRN Hypervisor:**
  The ACRN hypervisor is a type 1 hypervisor, running directly on the bare-metal hardware, and is suitable for a variety of IoT and embedded device solutions. It fetches and analyzes the guest instruction, then put the decoded information into the shared page as IOREQ, and notify/interrupt the VHM moulde in SOS to process.

- **VHM Module:**
  VHM is the abbreviation of Virtio and Hypervisor Service Module which is a kernel module in the Service OS (SOS) acting as a middle layer to support the device model and hypervisor. The VHM forwards the IOREQ to virtio-net backend driver to process.

- **ACRN Device Model / virtio-net Backend Driver:**
  The ACRN Device Model (DM) gets IOREQ from shared page and calls virtio-net backend driver to process. The backend driver receives the data in shared virtqueues and sends it to TAP device.

- **Bridge / Tap Device:**
  Bridge and Tap are standard virtual network infrastructures. They play an important role in communication among SOS, UOS and outside world.  

- **IGB Driver:**
  IGB is the physical NIC Linux kernel driver which is responsible for sending the data to the physical NIC.

<br>

The virtual network card (NIC) is implemented as a virtio legacy device in the ACRN device model (DM). It is registered as a PCI virtio device to the guest OS (UOS) and uses standard virtio-net in Linux kernel as its driver (the guest kernel should be built with `CONFIG_VIRTIO_NET=y`). The `virtio-net` backend in DM forward the data received from frontend to TAP device, then from TAP device to the bridge and finally from bridge to the physical NIC driver, e.g. IGB, vice versa.

<br>
# ACRN Network TX/RX Flow
<br>

Various components of ACRN network virtualization are illustrated in above architecture. Next, we will use UOS data transmission (TX) and reception (RX) as an example to explain step by step how these components are implemented and how they are working together. As you can see, this involves lots of modules, which is a good example for you to understand the virtualization in ACRN.

<br>
### **Initialization in Device Model**
<br>
**virtio_net_init**
-	Present frontend a virtual PCI based NIC
-	Setup control plan callbacks
-	Setup data plan callbacks, including TX, RX
-	Setup tap backend

<br>
### **Initialization in virtio-net Frontend Driver**
<br>


**virtio_pci_probe**
-	Construct virtio device using virtual pci device and register it to virtio bus

**virtio_dev_probe** --> **virtnet_probe** --> **init_vqs**
-	Register network driver
-	Setup shared virtqueues

<br>
### **ACRN UOS TX FLOW**
<br>

Following is the acrn UOS network TX flow, using TCP as an example.

<br>
**UOS TCP Layer**
```
tcp_sendmsg --> 
    tcp_sendmsg_locked --> 
        tcp_push_one --> 
            tcp_write_xmit --> 
                tcp_transmit_skb -->
```
<br>
**UOS IP Layer**
```
ip_queue_xmit --> 
    ip_local_out -->
        __ip_local_out -->
            dst_output -->
                ip_output -->
                    ip_finish_output -->
                        ip_finish_output2 -->
                            neigh_output -->
                                neigh_resolve_output -->
```
<br>
**UOS MAC Layer**
```
dev_queue_xmit -->
    __dev_queue_xmit --> 
        dev_hard_start_xmit --> 
            xmit_one -->  
                netdev_start_xmit -->
                    __netdev_start_xmit -->
```
<br>
**UOS MAC Layer `virtio-net` Frontend Driver**
```
start_xmit -->                   // virtual NIC driver xmit in virtio_net
    xmit_skb -->
        virtqueue_add_outbuf --> // add out buffer to shared virtqueue
            virtqueue_add -->
    virtqueue_kick -->           // notify the backend
        virtqueue_notify -->
            vp_notify -->
                iowrite16 -->    // trap here, HV will first get notified
```
<br>
**ACRN Hypervisor**
```
vmexit_handler -->                      // vmexit because VMX_EXIT_REASON_IO_INSTRUCTION
    pio_instr_vmexit_handler -->
        emulate_io -->                  // ioreq can’t be processed in HV, forward it to VHM
            acrn_insert_request_wait -->
                fire_vhm_interrupt -->  // interrupt SOS, VHM will get notified
```
<br>
**VHM Module**
```
vhm_intr_handler -->                          // VHM interrupt handler
    tasklet_schedule -->
        io_req_tasklet -->
            acrn_ioreq_distribute_request --> // ioreq can’t be processed in VHM, forward it to device DM
                acrn_ioreq_notify_client -->
                    wake_up_interruptible --> // wake up DM to handle ioreq
```
<br>
**ACRN Device Model / `virtio-net` Backend Driver**
```
handle_vmexit -->
    vmexit_inout -->
        emulate_inout --> 
            pci_emul_io_handler -->
                virtio_pci_write -->
                    virtio_pci_legacy_write -->
                        virtio_net_ping_txq -->       // start TX thread to process, notify thread return
                            virtio_net_tx_thread -->  // this is TX thread
                                virtio_net_proctx --> // call corresponding backend (tap) to process
                                    virtio_net_tap_tx -->
                                        writev -->    // write data to tap device
```
<br>
**SOS TAP Device Forwarding**
``` 
do_writev -->
    vfs_writev -->
        do_iter_write -->
            do_iter_readv_writev -->
                call_write_iter -->
                    tun_chr_write_iter -->
                        tun_get_user --> 
                            netif_receive_skb -->
                                netif_receive_skb_internal -->
                                    __netif_receive_skb -->
                                        __netif_receive_skb_core -->
```
<br>
**SOS Bridge Forwarding**
```
br_handle_frame -->
    br_handle_frame_finish -->
        br_forward -->
            __br_forward -->
                br_forward_finish --> 
                    br_dev_queue_push_xmit --> 
```
<br>
**SOS MAC Layer**                       
```
dev_queue_xmit -->
    __dev_queue_xmit --> 
        dev_hard_start_xmit --> 
            xmit_one -->  
                netdev_start_xmit --> 
                    __netdev_start_xmit -->
```
<br>
**SOS MAC Layer IGB Driver**
```
igb_xmit_frame --> // IGB physical NIC driver xmit function
```

<br>
### **ACRN UOS RX FLOW**
<br>

Following is the ACRN UOS network RX flow, using TCP as an example. Let’s start by receiving a device interrupt. 
(*\* Note that hypervisor will first get notified when receiving an interrupt even in passthrough cases.*)

<br>
**Hypervisor Interrupt Dispatch**
```
vmexit_handler -->                                    // vmexit because VMX_EXIT_REASON_EXTERNAL_INTERRUPT
    external_interrupt_vmexit_handler -->
        dispatch_interrupt -->
            common_handler_edge -->
	            ptdev_interrupt_handler -->
		            ptdev_enqueue_softirq --> // Interrupt will be delivered in bottom-half softirq
```
<br>
**Hypervisor Interrupt Injection**
```
do_softirq -->
    ptdev_softirq -->
        vlapic_intr_msi -->     // insert the interrupt into SOS
start_vcpu -->                  // VM Entry here, will process the pending interrupts
```
<br>
**SOS MAC Layer IGB Driver**
```
do_IRQ -->
    …
    igb_msix_ring -->
        igbpoll -->
            napi_gro_receive -->
                napi_skb_finish -->
                    netif_receive_skb_internal -->
                        __netif_receive_skb -->
                            __netif_receive_skb_core -->
```
<br>
**SOS Bridge Forwarding**
```
br_handle_frame -->
    br_handle_frame_finish -->
        br_forward -->
            __br_forward -->
                br_forward_finish -->
                    br_dev_queue_push_xmit -->
```
<br>
**SOS MAC Layer**
```
dev_queue_xmit -->
    __dev_queue_xmit -->
        dev_hard_start_xmit --> 
            xmit_one -->
                netdev_start_xmit -->
                    __netdev_start_xmit -->
```
<br>
**SOS MAC Layer TAP Driver**
```
tun_net_xmit --> // Notify and wake up reader process
```
<br>
**ACRN Device Model / `virtio-net` Backend Driver**
```
virtio_net_rx_callback -->       // the tap fd get notified and this function invoked
    virtio_net_tap_rx -->        // read data from tap, prepare virtqueue, insert interrupt into the UOS
        vq_endchains -->
            pci_generate_msi -->
```
<br>
**VHM Module**
```
vhm_dev_ioctl -->                // process the IOCTL and call hypercall to inject interrupt
    hcall_inject_msi -->
```
<br>
**ACRN Hypervisor**
```
vmexit_handler -->               // vmexit because VMX_EXIT_REASON_VMCALL
    vmcall_vmexit_handler -->
        hcall_inject_msi -->     // insert interrupt into UOS
            vlapic_intr_msi -->
```
<br>
**UOS MAC Layer `virtio-net` Frontend Driver**
```
vring_interrupt -->              // virtio-net frontend driver interrupt handler
    skb_recv_done -->            //registed by virtnet_probe-->init_vqs-->virtnet_find_vqs
        virtqueue_napi_schedule -->
            __napi_schedule -->
                virtnet_poll -->
                    virtnet_receive -->
                        receive_buf -->
```
<br>
**UOS MAC Layer**
```
napi_gro_receive -->
    napi_skb_finish -->
        netif_receive_skb_internal -->
            __netif_receive_skb -->
                __netif_receive_skb_core -->
```
<br>
**UOS IP Layer**
```
ip_rcv -->
    ip_rcv_finish -->
        dst_input -->
            ip_local_deliver -->
                ip_local_deliver_finish -->
```
<br>
**UOS TCP Layer**
```
tcp_v4_rcv -->
    tcp_v4_do_rcv -->
        tcp_rcv_established -->
            tcp_data_queue -->
                tcp_queue_rcv -->
                    __skb_queue_tail -->
                sk->sk_data_ready --> // application will get notified
```

<br>
# Usage
<br>

Following network infrastructure need to be prepared in SOS before we start. We need to create a bridge and at least one tap device (PS, two tap devices are needed if you want to create dual virtual NIC) and attach physical NIC and tap device to the bridge.

<br>

![vnet-structure](/assets/images/acrn-vnet/vnet-structure.png)
<p align="center">Figure 2: Network Infrastructure in SOS</p>

<br>

You can use Linux commands (e.g. `ip`, `brctl`) to create above network. In our case, we use systemd to automatically create the network by default. You can check the files with prefix `50-` in SOS `/usr/lib/systemd/network/`
> [50-acrn.netdev](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn.netdev) <br> [50-acrn.network](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn.network) <br> [50-acrn_tap0.netdev](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn_tap0.netdev) <br> [50-eth.network](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/eth.network)

<br>
When SOS is started, run `ifconfig` you will find devices created by above systemd configuration
```
acrn-br0  Link encap:Ethernet  HWaddr B2:50:41:FE:F7:A3
          inet addr:10.239.154.43  Bcast:10.239.154.255  Mask:255.255.255.0
          inet6 addr: fe80::b050:41ff:fefe:f7a3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:226932 errors:0 dropped:21383 overruns:0 frame:0
          TX packets:14816 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:100457754 (95.8 Mb)  TX bytes:83481244 (79.6 Mb)

acrn_tap0 Link encap:Ethernet  HWaddr F6:A7:7E:52:50:C6
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)

enp3s0    Link encap:Ethernet  HWaddr 98:4F:EE:14:5B:74
          inet6 addr: fe80::9a4f:eeff:fe14:5b74/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:279174 errors:0 dropped:0 overruns:0 frame:0
          TX packets:69923 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:107312294 (102.3 Mb)  TX bytes:87117507 (83.0 Mb)
          Memory:82200000-8227ffff

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:16 errors:0 dropped:0 overruns:0 frame:0
          TX packets:16 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1216 (1.1 Kb)  TX bytes:1216 (1.1 Kb)
```

<br>
Run `brctl show` to see the bridge `acrn-br0` and devices attached to it
```
bridge name     bridge id               STP enabled     interfaces
acrn-br0        8000.b25041fef7a3       no              acrn_tap0
                                                        enp3s0
```

<br>
Add a pci slot to the device model `acrn-dm` command line (mac address is optional)

> `-s 4,virtio-net,<tap_name>,[mac=<XX:XX:XX:XX:XX:XX>]`

<br>
when UOS is lauched, run `ifconfig` to check the network. `enp0s4` is the virtual NIC created by `acrn-dm`
```
enp0s4    Link encap:Ethernet  HWaddr 00:16:3E:39:0F:CD
          inet addr:10.239.154.186  Bcast:10.239.154.255  Mask:255.255.255.0
          inet6 addr: fe80::216:3eff:fe39:fcd/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:140 errors:0 dropped:8 overruns:0 frame:0
          TX packets:46 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:110727 (108.1 Kb)  TX bytes:4474 (4.3 Kb)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
```

<br>
# Conclusion
<br>

In this article, we introduce the network virtualization solution in ACRN, from the top level architecture to the detailed TX and RX flow. Currently, the control plane and data plane are all processed in ACRN device model, which may bring some overhead. But this is not a bottleneck at all for 1000Mbit NICs or below. The network bandwidth of virtualization can be very close to the native. For high speed NIC e.g. 10Gb or above, it is necessary to separate the data plane from the control plane. We can use vhost for acceleration. For most IoT scenarios, all in user space is simple and reasonable.

<br>

<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-124279546-2"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-124279546-2');
</script>
