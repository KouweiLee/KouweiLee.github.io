---
layout: post
title:  "Linux virtio driver与device间的通信"
date:   2024-1-8
categories: Linux
---

本文主要聚焦于armv8体系结构下linux virtio driver的初始化以及与qemu device的通信过程. 

## linux virtio driver的初始化

初始化virtio driver的过程中, 依次调用了如下几个重要的函数.

* virtio_mmio_probe

位于virtio_mmio.c, 探测virtio 设备, 并初始化写入VIRTIO_CONFIG_S_ACKNOWLEDGE表示发现了该设备.

调用了device_add(位于core.c), 不太重要.

* virtio_dev_probe

位于virtio.c, 使用对应的driver来初始化该设备. 首先标记VIRTIO_CONFIG_S_DRIVER, 表明找到合适的driver. 又设置VIRTIO_CONFIG_S_FEATURES_OK.

而后调用了virtio_driver的probe函数, 对于virtio_blk, 则是virtblk_probe.

该函数结束时, virtio设备的状态应该是VIRTIO_CONFIG_S_DRIVER_OK

* virtblk_probe

设置seg_max等属性.

调用init_vq-->

-->virtio_find_vqs: 设定该vq的callback函数为virtblk_done, 当设备完成IO操作后发中断时, 中断处理函数会调用该callback函数. 

-->vm_find_vqs: 注册了IO请求完成后中断处理函数为vm_interrupt. 

​	vm_setup_vq: 设置vq配置空间, 

​		vring_create_virtqueue_split: **设置vq->vq.callback为virtblk_done,vq->notify为vm_notify.** 并设置desc table, 已用环可用环								给device

之后再进行一些feature的协商, 并标记VIRTIO_CONFIG_S_DRIVER_OK.

## IO请求流程

### driver发出请求

> driver从Block层向底层的virtio driver派发请求的过程是一个异步的过程, 可能连续发出多个请求. 待device处理完请求后, 会调用相应的回调函数blk_mq_complete_request

blk_mq_dispatch_rq_list-->其中调用queue_rq, 此前的具体流程请看: [Linux block 层详解](https://zhuanlan.zhihu.com/p/501198341)

virtio_queue_rq-->

virtblk_add_req-->

virtqueue_add_sgs-->

virtqueue_add:向descriptor table添加一个描述符链, 并写入可用环

-->virtqueue_kick_prepare_split: 如果已用环的flags中有VRING_USED_F_NO_NOTIFY, 则表明设备不想要driver发出新的请求, driver则不发.

-->virtqueue_notify: 调用vm_notify, 写MMIO配置空间的notify寄存器

### device收到请求

对于qemu中的blk设备, 收到请求会首先进入virtio_blk_handle_output-->

virtio_blk_handle_vq-->

--> 从可用环中依次取出描述符链, 并形成IO向量. 

--> 将这些IO向量提交

-->virtio_blk_submit_multireq

待完成请求后, 通过中断注入的方式通知driver

### driver收到中断

中断处理函数:

vm_interrupt-->

vring_interrupt-->检查已用环是否变化, 并调用vq的callback

virtblk_done: 之后会根据已用环来回收描述符, 同时调用给block层的回调函数blk_mq_complete_request, 通知block层已经完成数据请求

## 参考资料

1. QEMU/KVM源码解析与应用, 机械工业出版社
2. [Linux block 层详解](https://zhuanlan.zhihu.com/p/501198341)
3. [linux kernel官方文档](https://www.kernel.org/doc/html/latest/block/blk-mq.html)