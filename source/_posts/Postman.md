---
title: Hack The Box - Postman の Write-Up
date: 2020-01-02 23:35:10
categories:
- WriteUPs
- Hack The Box
tags:
- Web
---
## 前言

&emsp;&emsp;元旦这两天闲来无事玩了下[Postman](https://www.hackthebox.eu/home/machines/profile/215)这台靶机，这是一台有很多坑的机器，不过总体难度不高。

![](/img/Postman/Postman1.png)

<!-- more -->

## 扫描

&emsp;&emsp;第一步还是常规的Nmap扫描：

![](/img/Postman/Postman2.png)

&emsp;&emsp;可以看到靶机开放了22，80和10000三个端口。22端口的ssh服务和80端口的web服务平淡无奇，简单看了下就略过了。

&emsp;&emsp;10000端口webmin服务吸引了我全部的注意，经过搜索发现webmin 1.910存在远程命令执行漏洞，CVE编号为CVE-2019-15107。但是经过多次尝试发现该漏洞无法利用，可能是被修补了，所以只能重新寻找突破口。

## 渗透

&emsp;&emsp;没办法，只能全端口重新扫描一下，看看有没有什么遗漏：

![](/img/Postman/Postman3.png)

&emsp;&emsp;果然6379端口隐藏了一个redis 4.0.9服务。找一下这个服务有没有什么可以利用的地方：

![](/img/Postman/Postman4.png)

&emsp;&emsp;果然redis 4.x/5.x版本也存在远程命令执行漏洞，但是这个脚本并不能使用。在不懈搜寻之下，找到了一些有用的东西：[https://github.com/Avinash-acid/Redis-Server-Exploit](https://github.com/Avinash-acid/Redis-Server-Exploit)。

&emsp;&emsp;这是[redis未授权访问的漏洞](https://xz.aliyun.com/t/4051)，通过该漏洞可以向靶机里写入ssh秘钥，然后通过ssh连接到靶机。

![](/img/Postman/Postman5.png)

&emsp;&emsp;这样我们便通过ssh以redis这个低权限账号登录。登录之后找了许多文件夹：

![](/img/Postman/Postman6.png)

&emsp;&emsp;在.bash_history文件中发现了Matt用户以及id_rsa.bak文件，找一下这个文件在哪里：

![](/img/Postman/Postman7.png)

&emsp;&emsp;猜测这是Matt用户的ssh登录凭证，同样使用john爆破密码，具体操作参考[Traverxec](https://coldwave96.github.io/2019/12/17/Traverxec/)，最终可以得到密码。

&emsp;&emsp;不过这次无法通过ssh密钥登陆，可能是靶机做了限制。然后我直接尝试了下用这个密码su Matt账户就成功切换到了Matt账户，然后在根目录下就可以获得user.txt。

## 提权

&emsp;&emsp;使用[LinEnum脚本](https://github.com/rebootuser/LinEnum)提权，即可在root根目录下找到root.txt。

&emsp;&emsp;其实root权限还有其他方法，还记得10000端口的webmin服务么，Matt用户以及刚才破解出来的密码配MSF中的webmin漏洞利用模块即可获取root用户权限的shell，然后在根目录下即可找到root.txt。