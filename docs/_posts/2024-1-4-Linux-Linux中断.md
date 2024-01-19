---
layout: post
title:  "Linux中断"
date:   2024-1-4
categories: Linux
---
## 硬件中断号与内核中断号的映射

Linux中, 注册中断接口函数`request_irq`使用的是Linux内核软件中断号, 又称IRQ号, 而不是GIC当中的硬件中断号. 

当Linux解析设备树时, 会将硬件中断号映射为IRQ号, 映射的过程是通过位图实现的, 也就是说这个映射的过程是动态进行的.

##  注册中断
