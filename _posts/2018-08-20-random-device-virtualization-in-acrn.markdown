---
layout: post
title:  "Random Device Virtualization in ACRN"
categories: ACRN
tags: ACRN
author: Jie Deng
description: 
---

# Introduction 
<br>

virtio-rnd is designed to provide a hardware random source for UOS. The virtual random device is based on virtio user mode framework. It simulates a PCI device based on virtio specification. Please refer to below ACRN hypervisor introduction and Virtio high level design for virtualization background introduction.

> [Introduction to Project ACRN](https://projectacrn.github.io/latest/introduction/index.html) <br>
[Virtio high-level design in ACRN](https://projectacrn.github.io/latest/developer-guides/virtio-hld.html)

<br>

# Architecture of Random Device Virtualization
<br>

Figure 1 shows the Random Device Virtualization Architecture in ACRN. The green components are parts of ACRN solution while the gray components are parts of Linux kernel. virtio-rnd is implemented as a virtio legacy device in the ACRN device model (DM), and which is registered as a PCI virtio device to the guest OS (UOS). Tools like `od` can be used to read randomness from /dev/random. This device file in UOS is bound with frontend virtio-rng driver (The guest kernel should be built with `CONFIG_HW_RANDOM_VIRTIO=y`). The backend virtio-rnd reads the HW randomness from /dev/random in SOS and sends them to frontend.

<br>

![random_architecture](/assets/images/acrn-vtrnd/random_architecture.png)
<p align="center">Figure 1: Random Device Virtualization in ACRN</p>

<br>

# How to Use

<br>
Add a pci slot to the device model `acrn-dm` command line, e.g.

> `-s 9,virtio-rnd`

<br>
Check if frontend virtio_rng driver is one of available random number generator drivers in UOS

> `# cat /sys/class/misc/hw_random/rng_available` <br> virtio_rng.0

<br>
Check if frontend virtio_rng currently connected to `/dev/random`
> `# cat /sys/class/misc/hw_random/rng_current` <br> virtio_rng.0

<br>
Get the randomness (*Note that HW randomness is precious resource of the system. The `od` will block and wait until there is randomness available*)
> `# od /dev/random`
