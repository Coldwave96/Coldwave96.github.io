---
title: 关于堆的bin结构的理解
date: 2020-07-02 15:18:20
categories:
- Theories
- Assembler
tags:
- PWN
- Heap
---
## 概述

用户申请堆空间是通过malloc函数实现的，而malloc在内核中对应的是ptmalloc。ptmalloc像是内核和用户的中间商，中间商向卖家（操作系统内核）要大量的货物（大块内存空间），然后分配给买家（用户程序）。同样的，当用户释放chunk的时候，也不是直接归还给系统，而是被ptmalloc所管理。当用户在一次请求内存空间的时候，ptmalloc会在空闲的chunk中选择合适大小的分配给用户。这种机制的好处在于可以避免频繁的系统效用，降低内存分配的开销，提高效率。

而ptmalloc正是通过bin结构来管理空闲堆块。它会根据空闲chunk的大小及使用状态将chunk分为4类：fast bin，small bin，large bin和unsorted bin。

<!-- more -->

## fast bin

对于size较小的chunk，释放之后单独处理，被放入fast bin中。

* 32位系统，fast bin中的chunk大小范围在16字节到64字节；

* 64位系统，fast bin中的chunk大小范围在32字节到128字节。

fast bin链表采用单向链表进行连接，并且每个bin采取了LIFO策略，最近释放的chunk会被更早地分配。所以当用户申请的chunk大小在fast bin范围内时，ptmalloc会首先判断fast bin中是否有对应大小的空闲chunk，有的话就会直接分配出去。

fast bin范围内chunk的inuse标志位始终被置为1，即它们不会和其他被释放的chunk合并，也就不会触发Unlink操作。

fastbin链表最末端的块fd域为0，此后每个块的fd域指向前一个块。因此通过fastbin只能泄漏heap的基地址。

## small bin

small bin中chunk大小范围的序号index从2到63，总共62个循环双向链表。

每个chunk的size大小用一个关系式表达是：`chunk_size = 2 * SIZE_SZ（4 or 8）* index`

从关系式可以看出每个链表中的chunk大小是一样的，这也是堆中堆表的数据结构。并且可以发现small bin中的chunk_size与fast bin有重复，这仅是大小重复，而不是chunk重复。fast bin中的chunk也有可能被放到small bin中去。

此外small bin中每个bin对应的链表采用FIFO策略，所以同一个链表中先被释放的chunk会被先分配。

通过smallbin可以获得：

* 1.libc.so的基地址；

* 2.heap基地址。

## large bin

large bin也是遵循FIFO策略的循环双向链表，一共有63个bin，每个bin中的chunk大小不一致，但处于一定区间范围内。32位系统中chunk_size >= 512字节。

## unsorted bin

unsorted bin可以看作空闲chunk回归其所属bin之前的缓冲区，该bin只有一个遵循FIFO策略的循环双向链表，且其中的free chunk处于乱序状态。unsorted bin暂时存储free后的chunk，一段时间后会将chunk放入对应的bin中去。

通过unsorted bin我们可以获取到某个堆块的地址和main_areana的地址。一旦获取到某个堆块的地址就可以通过malloc的size进行计算从而获得堆基地址。一旦获取到main_arena的地址，因为main_arena存在于libc.so中就可以计算偏移得出libc.so的基地址。

因此，通过unsorted bin可以获得：

* 1.libc.so的基地址；

* 2.heap基地址。

## PS

可以实现泄露的漏洞：

* 堆内存未初始化

* 堆溢出

* Use-After-Free

* 越界读

* heap extend
