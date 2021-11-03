---
title: Vulnhub - Raven 2 の Write-Up
date: 2020-05-26 17:05:08
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- Raven Series
---
## 前言

&emsp;&emsp;[Vulnhub](https://www.vulnhub.com/)之Raven系列靶机第二台[Raven 2]((https://www.vulnhub.com/entry/raven-2,269/))的Write-up。

<!-- more -->

## Step 1

&emsp;&emsp;寻找靶机地址：

![](/img/Raven-2/Raven-2-1.png)

&emsp;&emsp;扫描开放的端口和服务：

![](/img/Raven-2/Raven-2-2.png)

&emsp;&emsp;可以看到端口和服务与Raven 1一致。

## Step 2

&emsp;&emsp;尝试了之前Raven 1的手段，发现均失败，那只能另辟蹊径。

&emsp;&emsp;首先进行了目录爆破，然后去一个一个的访问界面寻找信息，果然找到一个有趣的文件夹：

![](/img/Raven-2/Raven-2-3.png)

&emsp;&emsp;在PATH文件里不仅找到了网站的实际地址，还找到了第一个flag：

![](/img/Raven-2/Raven-2-4.png)

&emsp;&emsp;看来这个文件夹里有重要的线索，仔细看了下，其他文件都指向PHPMailer。这时突然想起在Raven 1靶机里发现的那封奇怪的邮件和奇怪的code片段，基本确定Raven 2靶机的突破口就在这里了。

&emsp;&emsp;打开SECURITY.MD文件：

![](/img/Raven-2/Raven-2-5.png)

&emsp;&emsp;这是连利用哪个漏洞都告诉我们了，结合在VERSION中找到的版本号：

![](/img/Raven-2/Raven-2-6.png)

&emsp;&emsp;所以就锁定了[CVE-2016-10033](https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2016-10033)，还是最危险的远程代码执行漏洞。

## Step 3

&emsp;&emsp;关于这个漏洞的具体信息可以看这篇红日安全团队的文章[Web安全-CVE-2016-10033漏洞](https://www.freebuf.com/column/155911.html)。

&emsp;&emsp;找一下利用的EXP：

![](/img/Raven-2/Raven-2-7.png)

&emsp;&emsp;在Raven 1靶机提权的时候我们用的就是Python实现的，所以这里也选择Python版本的EXP。

&emsp;&emsp;我们看一下这个EXP，然后针对本次环境需要修改的参数：target即目标地址，反弹回本机的地址以及Webshell上传的地址（P.S.backdoor的文件名一定要修改，否则无法攻击成功）

![](/img/Raven-2/Raven-2-8.png)

&emsp;&emsp;改好之后运行脚本，然后监听本机4444端口，访问Webshell触发shell反弹：

![](/img/Raven-2/Raven-2-9.png)

&emsp;&emsp;得到反弹的shell之后利用Python转换成交互式shell，看下文件夹，发现shell.php确实是成功上传了，但是没有找到backdoor.php，具体原因也不太清楚。

![](/img/Raven-2/Raven-2-10.png)

&emsp;&emsp;然后找一下有没有flag：`find / -name "flag*”`

![](/img/Raven-2/Raven-2-11.png)

&emsp;&emsp;找到flag2.txt：

![](/img/Raven-2/Raven-2-12.png)

&emsp;&emsp;然后是flag3.png文件：

![](/img/Raven-2/Raven-2-13.png)

## Step 4

&emsp;&emsp;根据Raven 1的经验，最后一个flag需要拿到root权限。提权的方法和Raven 1补充的MySQL udf提权一样。

&emsp;&emsp;拿到shell之后查看wordpress配置文件得到数据库账号和密码，登录数据库通过udf提权即可获取root权限。

&emsp;&emsp;这一步与Raven 1靶机有两点不同：

* 一是没法直接通过scp命令将文件传入靶机，需要靶机通过wget命令下载攻击机的文件

* 二是数据库里用户的账户密码不再能够通过ssh登录了，所以Raven 1中通过steven账户sudo命令提权这条路走不通了

&emsp;&emsp;具体操作见Raven 1即可。
