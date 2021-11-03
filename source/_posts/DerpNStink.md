---
title: Vulnhub - DerpNStink の Write-Up
date: 2020-06-08 11:48:02
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- DC Series
---
## 前言

&emsp;&emsp;[Vulnhub](https://www.vulnhub.com/)之[DerpNStink](https://www.vulnhub.com/entry/derpnstink-1,221/)的Write-up。

<!-- more -->

## Step 1

&emsp;&emsp;寻找靶机地址：

![](/img/DerpNStink/DerpNStink1.png)

&emsp;&emsp;扫描开放的端口和服务：

![](/img/DerpNStink/DerpNStink2.png)

## Step 2

&emsp;&emsp;21端口开放了vsftpd服务，之前这个服务是存在远程命令执行漏洞的，不过想要利用这个漏洞需要获得一个账号。

&emsp;&emsp;除此之外还有22端口的ssh服务和80端口的http服务，先从http服务开始：

![](/img/DerpNStink/DerpNStink3.png)

&emsp;&emsp;主页比较可爱，查看源码在最下面找到flag1:

![](/img/DerpNStink/DerpNStink4.png)

## Step 3

&emsp;&emsp;扫描网站目录发现一个比较奇特的地址：

![](/img/DerpNStink/DerpNStink5.png)

&emsp;&emsp;访问一下发现下面的界面信息：

![](/img/DerpNStink/DerpNStink6.png)

![](/img/DerpNStink/DerpNStink7.png)

&emsp;&emsp;根据提示在hosts文件里添加本地dns：`172.16.83.139 derpnstink.local`。然后再次进行目录爆破：

![](/img/DerpNStink/DerpNStink8.png)

&emsp;&emsp;找到数据库界面的地址。还找到一个新的界面，如提示所说的WordPress搭建的Blog。

![](/img/DerpNStink/DerpNStink9.png)

![](/img/DerpNStink/DerpNStink10.png)

&emsp;&emsp;通过wpscan扫描发现了该站点使用了plugin：

![](/img/DerpNStink/DerpNStink11.png)

&emsp;&emsp;以及该站点的用户：

![](/img/DerpNStink/DerpNStink12.png)

## Step 4

&emsp;&emsp;一般wordpress靶机装了插件的，基本都会有插件漏洞，找了一下：

![](/img/DerpNStink/DerpNStink13.png)

&emsp;&emsp;果然，版本也完美匹配，应该就是这个漏洞了，去找一下利用工具：

![](/img/DerpNStink/DerpNStink14.png)

&emsp;&emsp;可是看下利用模块，需要一个wordpress的账户和密码：

![](/img/DerpNStink/DerpNStink15.png)

&emsp;&emsp;只能结合枚举出来的两个用户尝试爆破了：

![](/img/DerpNStink/DerpNStink16.png)

&emsp;&emsp;居然admin/admin……行吧。万事俱备，MSF走起：

![](/img/DerpNStink/DerpNStink17.png)

&emsp;&emsp;成功获得shell，尝试了下直接找flag文件没有找到，那只能看一些敏感文件获取更多的信息，在wp-config.php文件里找到数据库登录名和密码：

![](/img/DerpNStink/DerpNStink18.png)

&emsp;&emsp;结合之前目录爆破找到的phpmyadmin地址登录到数据库中，找到flag2：

![](/img/DerpNStink/DerpNStink19.png)

## Step 5

&emsp;&emsp;同时在user表中获得两个账户的密码：

![](/img/DerpNStink/DerpNStink20.png)

&emsp;&emsp;尝试破解密码：

![](/img/DerpNStink/DerpNStink21.png)

&emsp;&emsp;root破解出来当然就是admin，unclestinky密码破解出来是wedgie57：

![](/img/DerpNStink/DerpNStink22.png)

&emsp;&emsp;尝试了ssh发现不行，突然想起来还有个ftp服务，那么getshell？好像我们已经拿过shell了，也许是个高权限的shell？呵，发现3.0.4版本没有远程命令执行漏洞......尴尬，还以为自己早就未卜先知了==

&emsp;&emsp;那就正常登录看看有什么，用unclestinky账户登录一直失败，想起来webnotes/info.txt界面又个账户名叫stinky，异想天开的用这个试了一下，然后就进去了==

&emsp;&emsp;在里面找到一段对话：

![](/img/DerpNStink/DerpNStink23.png)

&emsp;&emsp;然后进了无数个ssh文件夹之后找到一个key.txt：

![](/img/DerpNStink/DerpNStink24.png)

&emsp;&emsp;ssh的登录凭证，总算找到了一点东西了。用这个凭证登录ssh：

![](/img/DerpNStink/DerpNStink25.png)

&emsp;&emsp;好吧，`chmod 600 key.txt`去掉其他用户的read权限后再次登陆：

![](/img/DerpNStink/DerpNStink26.png)

&emsp;&emsp;终于进来了，找下有什么东西，首先是flag3:

![](/img/DerpNStink/DerpNStink27.png)

&emsp;&emsp;还有一个pcap文件：

![](/img/DerpNStink/DerpNStink28.png)

## Step 6

&emsp;&emsp;下载到本地看下是什么：

![](/img/DerpNStink/DerpNStink29.png)

&emsp;&emsp;用wireshark打开发现是一段时间内的数据包，根据pcap包的名字结合之前在ftp里找到的那段对话，不难发现这段数据包中可能包含mrderp修改的账号和密码信息。

&emsp;&emsp;在pcap文件中寻找一番，终于发现了用户名和密码：

![](/img/DerpNStink/DerpNStink30.png)

&emsp;&emsp;经过尝试发现该用户名和密码可以登录ssh服务：

![](/img/DerpNStink/DerpNStink31.png)

&emsp;&emsp;通过`sudo -l`命令查看有什么可以利用的提权方式：

![](/img/DerpNStink/DerpNStink32.png)

&emsp;&emsp;得知mrderp用户可以通过sudo命令在/home/mrderp/binaries/文件下执行derpy*的命令。所以我们创建binaries文件夹：

![](/img/DerpNStink/DerpNStink33.png)

&emsp;&emsp;再创建derpy.sh文件，向里面写入：

![](/img/DerpNStink/DerpNStink34.png)

&emsp;&emsp;最后赋予该文件执行权限后通过sudo执行即可获取root权限：

![](/img/DerpNStink/DerpNStink35.png)

&emsp;&emsp;最终找到flag.txt文件：

![](/img/DerpNStink/DerpNStink36.png)

&emsp;&emsp;至此，该靶机4个flag已经全部找到。