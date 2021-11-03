---
title: Vulnhub - Lord Of The Root の Write-Up
date: 2020-06-03 16:18:10
categories:
- WriteUPs
- Vulnhub
tags:
- Web
---
## 前言

&emsp;&emsp;[Vulnhub](https://www.vulnhub.com/)之[Lord Of The Root](https://www.vulnhub.com/entry/lord-of-the-root-101,129/)的Write-up。

<!-- more -->

## Step 1

&emsp;&emsp;寻找靶机地址：

![](/img/LordOfTheRoot/LordOfTheRoot1.png)

&emsp;&emsp;扫描开放的端口和服务：

![](/img/LordOfTheRoot/LordOfTheRoot2.png)

&emsp;&emsp;扫描结果提示OSScan results may be unreliable because we could not find at least 1 open and 1 closed port，可能是靶机做了针对操作系统检测的防范。

## Step 2

&emsp;&emsp;扫描结果只有一个22端口，只能先尝试登录：

![](/img/LordOfTheRoot/LordOfTheRoot3.png)

&emsp;&emsp;ssh登录提示上面的界面，猜测是端口敲门，这点在[DC-9靶机](https://coldwave96.github.io/2020/05/06/DC-9/#Step-4)中也有涉及到。

&emsp;&emsp;所以我们通过`knock 172.16.83.138 1 2 3`命令实现端口敲门，然后再次扫描靶机开放的端口：

![](/img/LordOfTheRoot/LordOfTheRoot4.png)

&emsp;&emsp;果然多了一个1337端口，访问之后发现就是一张简单的图片：

![](/img/LordOfTheRoot/LordOfTheRoot5.png)

&emsp;&emsp;尝试下爆破网站目录，发现/images目录下有3张照片，robots.txt文件中有一段奇怪的字符串：

![](/img/LordOfTheRoot/LordOfTheRoot6.png)

&emsp;&emsp;这是base64加密的字符串，揭秘后得到：Lzk3ODM0NTIxMC9pbmRleC5waHA= Closer!

&emsp;&emsp;但是前半部分依然是base64加密的字符串，再次解密得到：/978345210/index.php

&emsp;&emsp;访问这个界面看到一个登录框：

![](/img/LordOfTheRoot/LordOfTheRoot7.png)

## Step 3

&emsp;&emsp;通过sqlmap的注入发现这里存在注入漏洞：

![](/img/LordOfTheRoot/LordOfTheRoot8.png)

&emsp;&emsp;在Webapp数据库里发现Users表：

![](/img/LordOfTheRoot/LordOfTheRoot9.png)

&emsp;&emsp;User表里找到5个用户的账户和密码:

![](/img/LordOfTheRoot/LordOfTheRoot10.png)

&emsp;&emsp;用这5个账户密码登录之前的界面，全部跳转到同一个profile.php界面，而这个界面不用输入任何账户名和密码直接点击登录就可以跳转，所以这5个账户名和密码不是用在这里的。

&emsp;&emsp;将这5个账户名和密码用于爆破22端口ssh服务：

![](/img/LordOfTheRoot/LordOfTheRoot11.png)

&emsp;&emsp;用找到的账户ssh登录成功：

![](/img/LordOfTheRoot/LordOfTheRoot12.png)

## Step 4

&emsp;&emsp;下一步就是提权了。使用sudo -l命令提示该用户没有sudo权限，这样SUID提权方法就无法使用了：

![](/img/LordOfTheRoot/LordOfTheRoot13.png)

&emsp;&emsp;找了下资料，大佬给了2个工具网站：

* [unix-privesc-check](http://pentestmonkey.net/tools/audit/unix-privesc-check)

* [linuxprivchecker](https://www.securitysift.com/download/linuxprivchecker.py)

&emsp;&emsp;这两个脚本将细致的检测许多可导致提权的配置问题，我们选一个尝试一下。将脚本下载到靶机上，然后运行：

![](/img/LordOfTheRoot/LordOfTheRoot14.png)

&emsp;&emsp;脚本最后反馈了一个很眼熟的信息：

![](/img/LordOfTheRoot/LordOfTheRoot15.png)

&emsp;&emsp;这不是MySQL的udf提权么，之前在[Raven1](https://coldwave96.github.io/2020/05/25/Raven-1/#P-S)中详细谈论过，步骤基本一致，有兴趣的可自行动手尝试。

&emsp;&emsp;关于MySQL数据库root账户的密码我们可以通过sqlmap在mysql数据库的user表中找到。
