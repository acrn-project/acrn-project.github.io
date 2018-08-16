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

The virtio-net is the paravirtualization solution used in ACRN. The ACRN device model emulate virtual NICs for UOS use and the frontend virtio network driver operate the virtual NIC. Both of them follow the virtio specification.

<br>

# Feathures
<br>
-	Legacy device supported, modern device not supported
-	Two virtqueues are used in virtio-net, i.e. RX queue and TX queue
-	Virtio indirect descriptor supported
-	TAP backend supported
-	Control queue not supported
-	NIC multiple queue not supported

<br>

# Architecture
<br>

Following figure 1 shows the network virtualization architecture in ACRN.

<br>

![network-virtualization-architecture](/assets/images/acrn-vnet/network-virtualization-architecture.png)
<p align="center">Figure 1: Network Virtualization Architecture</p>

<br>

There is a detailed explanation about the I/O emulation flow of ACRN, see [ACRN I/O mediator](https://projectacrn.github.io/latest/introduction/index.html#acrn-io-mediator). The virtio-net backend in DM forward the data received from frontend to TAP device, then from TAP device to the bridge and finally from bridge to the physical NIC driver, e.g. igb, vice versa.

virtio-net is implemented as a virtio legacy device in the ACRN device model (DM), and which is registered as a PCI virtio device to the guest OS (UOS). No changes are required in the frontend Linux virtio-net driver except that the guest kernel should be built with `CONFIG_VIRTIO_NET=y`.

<br>

# How to Use
<br>

We need to prepare following network infrastructure on SOS before we start. We need to create a bridge and at least one tap device (e.g. Two tap devices are needed if you want to create dual virtual NIC) and attach physical NIC and tap devices to the bridge.

<br>

![vnet-structure](/assets/images/acrn-vnet/vnet-structure.png)
<p align="center">Figure 2: Network Infrastructure in SOS</p>

<br>

You can use Linux commands to create above network. In our case, we use systemd to automatically create the network by default. You can check the files with prefix `50-` in SOS `/usr/lib/systemd/network/`
> [50-acrn.netdev](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn.netdev) <br> [50-acrn.network](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn.network) <br> [50-acrn_tap0.netdev](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/acrn_tap0.netdev) <br> [50-eth.network](https://raw.githubusercontent.com/projectacrn/acrn-hypervisor/master/tools/acrnbridge/eth.network)

Add a pci slot to the device model `acrn-dm` command line:

> `-s 4,virtio-net,<tap name>`

<br>
# Code Analysis
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
**Hypervisor**
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
**ACRN Device Model**
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
**SOS Physical NIC driver** 
```
igb_xmit_frame --> // igb physical NIC driver xmit function
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
**ACRN Device Model**
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
**Hypervisor**
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
# Conclusion
<br>

In this article, we introduce the network virtualization solution in ACRN, from the top level architecture to the detailed TX and RX flow. Currently, the control plane and data plane are all processed in ACRN device model, which may bring some overhead. But this is not a bottleneck at all for 1000Mbit NICs or below. The network bandwidth of virtualization can be very close to the native. For high speed NIC e.g. 10Gb or above, it is necessary to separate the data plane from the control plane. We can use vhost for acceleration. For most IoT scenarios, all in user space is simple and reasonable.