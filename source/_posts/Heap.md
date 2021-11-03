---
title: 堆溢出入门基础知识
date: 2020-06-30 15:08:39
categories:
- Theories
- Assembler
tags:
- PWN
- Heap
---
## 前言

&emsp;&emsp;这几天在看堆溢出相关的资料，这里简单做个小结。不是很全面，仅仅列出一些相对比较重要的内容。

<!-- more -->

## 堆的概念

&emsp;&emsp;堆是由程序员自行申请和释放的内存区块，分别通过malloc和free函数实现。根据Linux系统的内存管理机制，堆其实是程序虚拟地址空间的一块连续的线性区域。与栈相反，堆由内存低地址向高地址方向增长，但是由于两者起始地址相距很远，所以基本不会出现交汇的现象。

## 堆的结构

&emsp;&emsp;在程序的执行过程中，由malloc申请的内存称为chunk，这块内存在ptmalloc内部用malloc_chunk结构体表示。

&emsp;&emsp;一个malloc_chunk结构如下：

```C#
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
```

&emsp;&emsp;字段解释：

&emsp;&emsp;prev_size，如果该chunk的物理相邻的前一地址chunk（两个指针的地址差值为前一chunk大小）是空闲的话，那该字段记录的是前一个chunk的大小（包括chunk头）。否则，该字段可以用来存储物理相邻的前一个chunk的数据。

&emsp;&emsp;size，该chunk的大小，大小必须是2*SIZE_SZ的整数倍。32位系统中，SIZE_SZ是4；64位系统中，SIZE_SZ是8。

* 该字段的低三个比特位对chunk的大小没有影响，他们从高到低分别表示：

```Code
* NON_MAIN_ARENA，记录当前chunk是否不属于主线程，1表示不属于，0表示属于
* IS_MAPPED，记录当前chunk是否是由mmap分配的
* PREV_INUSE，记录前一个chunk块是否被分配，1表示被分配，0时可以通过prev_size字段来获取上一个chunk的大小及地址，这样方便进行空闲chunk之间的合并
```

&emsp;&emsp;fd，bk。fd指向下一个（非物理相邻）空闲的chunk，bk指向上一个（非物理相邻）空闲的chunk。

&emsp;&emsp;fd_nextsize，bk_nextsize，只有chunk空闲的时候才使用，不过其用于较大的chunk。fd_nextsize指向前一个与当前chunk大小不同的第一个空闲块，不包含bin的头指针。bk_nextsize指向后一个与当前chunk大小不同的第一个空闲块，不包含bin的头指针。

## 堆的操作

&emsp;&emsp;主要关注的是unlink，用来将一个双向链表（只存储空闲的chunk）中的一个元素取出来，应用场景有malloc、free、malloc_consolidate、realloc。

## 堆溢出

&emsp;&emsp;堆溢出是指程序向某个堆块中写入的字节数超过了堆块本身可使用的字节数（注意是可使用的字节数位不是用户申请的字节数，因为堆管理器会对用户所申请的字节数进行调整，这也导致可利用的字节数都不小于用户申请的字节数），因而导致了数据溢出，并覆盖到物理相邻的高地址的下一个堆块。

&emsp;&emsp;堆溢出策略有：

* 覆盖与其物理相邻的下一个chunk的内容。

* 利用堆中机制（如unlink等）来实现任意地址写入（Write-Anything-Anywhere）或控制堆块中的内容等效果，从而来控制程序的执行流。

&emsp;&emsp;堆溢出中的几个重要步骤：

* 寻找堆分配函数；

```Code
* malloc
* calloc，在分配后会自动进行清空
* realloc
```

* 寻找危险函数;

```Code
/* 输入 */
* gets，直接读取一行，忽略’\x00’
* scanf
* vscanf

/* 输出 */
* sprintf

/* 字符串 */
* strcpy，字符串复制，遇到’\x00’停止
* stract，字符串拼接，遇到’\x00’停止
* bcopy
```

* 确定填充长度：计算开始写入的地址与所要覆盖的地址之间的距离。
