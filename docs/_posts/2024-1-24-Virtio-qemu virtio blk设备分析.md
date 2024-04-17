---
layout: post
title:  "qemu virtio blk设备分析"
date:   2024-1-24
categories: 虚拟化
---

# qemu virtio blk设备分析

对于virtio-mmio类型的设备来说, 当virtio driver读写mmio区域时, 会发生vm exit, 进入kvm, kvm根据读写偏移进入qemu, qemu最终会调用virtio_mem_ops指定的函数:

```c
static const MemoryRegionOps virtio_mem_ops = {
    .read = virtio_mmio_read,   // 读调用的函数
    .write = virtio_mmio_write, // 写调用的函数
    .endianness = DEVICE_LITTLE_ENDIAN,
};
```

当virtio driver写mmio空间中的queue notify寄存器时, 即表示virtqueue可用环有数据, 要对virtio blk设备进行读写, 此时qemu会调用:

virtio_mmio_write-->

virtio_queue_notify: 调用virtqueue注册的handle_output函数, 对于virtio-blk则是virtio_blk_handle_output

virtio_blk_handle_output-->

virtio_blk_handle_vq: 在一个while循环中, 依次调用

* virtio_blk_get_request: 获取可用环中第一个描述符链表示的IO请求, 并转化成VirtIOBlockReq形式返回. 其中的virtqueue_map_desc函数会将描述符链中所有的guest physical address转换为qemu进程的虚拟地址空间的地址host virtual address. 
* virtio_blk_handle_request: 处理将上一个函数得到的VirtIOBlockReq, 对于读写请求, 则会将其加入到函数参数MultiReqBuffer mrb中
* virtio_blk_submit_multireq-->submit_requests-->blk_aio_preadv->blk_aio_prwv: 将所有IO请求打包成QEMUIOVector的形式, 并创建一个[qemu协程](https://blog.csdn.net/huang987246510/article/details/93139257), 协程执行的函数为blk_aio_read_entry. 

> 创建协程的原因: qemu的主体逻辑采用事件驱动的形式, 这样带来了大量异步函数操作, 每个异步函数又有一个回调函数, 每个回调函数参数里还得包含上下文信息, 导致代码逻辑支离破碎较为复杂. 由于协程只能在自我yield和退出时才会发生协程的切换, 且一个协程拥有自己独立的上下文和栈空间, 在一个函数中创建协程后, 当前函数的执行流被阻塞, 协程的执行流开始执行, 在协程中进行异步操作回调函数可以不包含上下文信息, 因为一个协程对应一个异步操作, 协程本身又是具有单独上下文的. 因此便于代码逻辑清晰. 

协程执行: 

blk_aio_read_entry

blk_co_do_preadv_part->(qemu 7.2.1)

bdrv_co_preadv_part->

bdrv_co_preadv->(drv->bdrv_co_preadv)

raw_co_preadv->(raw-format.c)

bdrv_co_preadv->

bdrv_co_preadv_part->

bdrv_aligned_preadv

bdrv_driver_preadv->

raw_co_preadv(file-posix.c)->

raw_co_prw: 由于当前qemu未开启CONFIG_LINUX_IO_URING和CONFIG_LINUX_AIO功能, 因此不进行异步IO的优化, 而是通过多线程的方法来提高并发度-->

raw_thread_pool_submit: 从线程池中创建一个线程, 该线程会执行handle_aiocb_rw函数. 

handle_aiocb_rw->

handle_aiocb_rw_linear-->

pread......(如果发现有多个缓冲区, 则使用preadv一次性进行读取, 减少系统调用次数) 

因此可以看到, 未采用aio和io_uring这些异步IO优化的virtio blk设备, 采用多线程+pread对文件进行读取, 模拟磁盘设备. 

以下是通过gdb调试qemu打印函数调用栈的截图:

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240124102400963.png" alt="image-20240124102400963" style="zoom:50%;" />

<img src="https://mdpics4lgw.oss-cn-beijing.aliyuncs.com/aliyun/image-20240124102424720.png" alt="image-20240124102424720" style="zoom:50%;" />

用gdb调试qemu的命令:

```bash
sudo gdb-multiarch --args qemu-system-aarch64 \
    -machine virt,gic_version=3 \
    -machine virtualization=true \
    -dtb hvisor.dtb \
    -cpu cortex-a57 \
    -machine type=virt \
    -nographic \
    -smp 4  \
    -m 2G \
    -kernel linux-Image \
    -append "console=ttyAMA0 root=/dev/vda rw mem=768m" \
    -drive if=none,file=/home/lgw/study/hypervisor/new_readme/ubuntu-20.04-rootfs_ext4.img,id=hd0,format=raw \
    -device virtio-blk-device,drive=hd0 \
    -net nic \
    -net user,hostfwd=tcp::2333-:22 \
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

## qemu的virtio blk中是否有DMA操作

代码中唯一出现有关DMA的代码, 是在address_space_map函数中, flatview_translate没有进行有关IOMMU的操作, fuzz_dma_read_cb则是个空函数, 用于: 这个函数的主要目的是在进行虚拟机模糊测试时，模拟 DMA 读取操作，生成模糊测试所需的数据，并记录 DMA 区域以避免重复获取。这样可以模拟 DMA 操作对虚拟机内存的影响，用于测试虚拟机的正确性和稳定性。

因此QEMU用软件实现的virtio blk，是不涉及硬件DMA的，因为它是通过CPU从内存中读写vring。但是，对于硬化在FPGA板子上的virtio后端设备，不是用软件实现的，它读写内存时就会通过DMA操作了。