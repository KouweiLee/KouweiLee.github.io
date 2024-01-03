---
layout: post
title:  "ScatterList基础"
date:   2024-1-3
categories: Linux
---
scatterlist, 全称物理内存散列表, 当存在一片连续的虚拟内存区域时, 由于该虚拟内存可能对应着多个非连续的物理内存区域, Linux内核使用scatterlist来表示表示va和pa之间的映射关系. 一个scatterlist结构体对象就表示其中的一片连续的物理内存区域:

```c
struct scatterlist {
	unsigned long	page_link; //该内存区域所在的虚拟页面地址。bit0和bit1有特殊用途（可参考后面的介绍），因此要求page最低4字节对齐
	unsigned int	offset; //指示该内存区域在页面中的偏移（起始位置）
	unsigned int	length; //该内存区域的长度。
	dma_addr_t	dma_address; // 内存区域的物理起始地址
#ifdef CONFIG_NEED_SG_DMA_LENGTH
	unsigned int	dma_length;
#endif
};
```

在实际的应用场景中，单个的scatterlist对象是没有多少意义的，我们需要多个scatterlist组成一个链表数组，以表示在物理上不连续的虚拟地址空间. kernel中可以用sg_table表示这种数组(也可以不用): 

```c
struct sg_table {
        struct scatterlist *sgl;        /* the list */
        unsigned int nents;             /* number of mapped entries */
        unsigned int orig_nents;        /* original size of list */
};
```

其中`sgl`表示一个scatterlist的数组, orig_nents为该数组的大小, nents为该数组中有效元素的个数. 

而scatterlist数组中到底有多少有效的scatterlist, 则是由以下2个规则确定:

1. 如果scatterlist数组中某个scatterlist的page_link的bit0为1，表示该scatterlist不是一个有效的内存块，而是一个chain（铰链），指向另一个scatterlist数组。通过这种机制，可以将不同的scatterlist数组链在一起，因为scatterlist也称作chain scatterlist。

2. 如果scatterlist数组中某个scatterlist的page_link的bit1为1，表示该scatterlist是scatterlist数组中最后一个有效内存块（后面的就忽略不计了）。

参考资料: http://www.wowotech.net/memory_management/scatterlist.html