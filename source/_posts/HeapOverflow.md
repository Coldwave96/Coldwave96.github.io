---
title: 堆溢出漏洞小结
date: 2020-07-07 16:49:57
categories:
- Theories
- Assembler
tags:
- PWN
- Heap
---
## 概述

总结一些堆溢出漏洞利用姿势。

<!-- more -->

## 姿势1：Off By One

Off By One指程序向缓冲区中写入时，写入的字节数超过程序申请的字节数并且只越界一个字节。

出现这种漏洞主要有两个原因：

* 边界验证不严，例如使用循环语句向堆块中写入数据时，循环次数设置错误导致多写入了一个字节；

* 字符串操作不合适，例如strlen函数计算字符串长度时是不考虑结束符’\x00’的，而strcpy复制字符串的时候会拷贝结束符’\x00’，函数使用不当会导致向堆中多写入一个字。

## 姿势2：Unsorted Bin Attack

Unsorted Bin Attack攻击实现的前提是控制Unsorted Bin Chunk的bk指针，利用该漏洞可以实现修改任意地址值为一个较大的数值。

Unsorted Bin Attack攻击可以实现修改任意地址的值，但是所修改成的值却不受我们控制，唯一可以确定的是这个值比较大。

通过这个漏洞我们可以实现：

* 修改循环次数使得程序可以执行多次循环；

* 修改heap中的global_max_fast使得更大的chunk被视为fast bin，然后实现Fastbin Attack。

## 姿势3：Use After Free

Use After Free字面意思是当一个内存块被释放之后被再次使用，实际情况有：

* 内存块被释放后，其对应的指针被设置为 NULL ， 然后再次使用，自然程序会崩溃。

* 内存块被释放后，其对应的指针没有被设置为 NULL ，然后在它下一次被使用之前，没有代码对这块内存块进行修改，那么程序很有可能可以正常运转。

* 内存块被释放后，其对应的指针没有被设置为NULL，但是在它下一次使用之前，有代码对这块内存进行了修改，那么当程序再次使用这块内存时，就很有可能会出现奇怪的问题。

Use After Free漏洞一般指后两种情况，被释放后没有被设置为NULL的内存指针一般被称为dangling pointer。

## 姿势4：Chunk Extend and Overlapping

通过Chunk Extend实现Chunk Overlapping。实现Chunk Extend需要程序中存在基于堆的漏洞，利用之后可以实现控制chunk header中的数据。

## 姿势5：Fastbin Attack

Fastbin Attack是指基于fastbin机制的一类漏洞利用方法，这类漏洞的利用前提是：

* 存在堆溢出、Use After Free等能控制chunk内容的漏洞；

* 漏洞发生于fastbin类型的chunk中。

细分可以分为：

* Fastbin Double Free

* House of Spirit

* Alloc to Stack

* Arbitrary Alloc

前两种主要漏洞侧重于利用free函数释放真的chunk或伪造的chunk，然后再次申请chunk进行攻击，后两种侧重于故意修改fd指针，直接利用malloc申请指定位置chunk进行攻击。

### Fastbin Double Free

Fastbin Double Free指fastbin中的chunk多次释放。根据fastbin的LIFO策略，多次分配的都是同一个堆块。这样相当于多个指针指向同一个堆块，结合该堆块的数据可以实现类型混淆的效果。

Fastbin Double Free漏洞利用原理：

* fastbin的堆块被释放后next_chunk的pre_inuse位不会清空；

* fastbin在free时仅验证main_arena直接指向的堆块，也即链表头部的堆块，而没有验证堆表后面的堆块。

当释放chunk1后，再释放chunk2，这时main_arena指向chunk2，所以可以再次释放chunk1。此时chunk1的fd仍旧指向chunk2且不会被清空，如果我们可以控制chunk1的数据，便可以写入fd指针实现指定任意地址分配fastbin堆块，这样就相当于实现了在任意地址写入任意值的效果。

### House of Spirit

House of Spirit是在目标位置处伪造fastbin chunk并将其释放，从而实现在指定地址处分配chunk。但是需要能够控制指定地址的内容，并在指定地址布局好fake chunk的结构且fake chunk需要满足堆的检验机制。

### Alloc to Stack

Alloc to Stack技术是劫持fastbin链表中chunk的fd指针，将该指针指向想要分配的栈上，从而实现控制栈中的关键数据，档案栈上需要存在有满足条件的size值。

### Arbitrary Alloc

Arbitrary Alloc技术和Alloc to Stack技术其实一样，只不过Arbitrary Alloc不再局限于栈中，只要任意的目标地址存在满足条件的size域即可。
