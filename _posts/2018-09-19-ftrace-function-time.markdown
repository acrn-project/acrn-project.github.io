---
layout: post
title:  "测量函数运行时间"
categories: ACRN
tags: ACRN
author: Jie Deng
description: 
---

1. 运行一个VBSK应用(数据采集时间保持运行)
2. 测量trap时间 
   > funcslower vp_notify 1
3. 测量nofiy时间
   > funcslower virtio_vq_endchains 1
4. 采集10000个左右的数据，计算平均值  

[下载funcslower](https://raw.githubusercontent.com/brendangregg/perf-tools/master/kernel/funcslower)