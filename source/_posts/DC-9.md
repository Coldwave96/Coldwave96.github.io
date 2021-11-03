---
title: Vulnhub - DC-9 の Write-Up
date: 2020-05-06 10:07:32
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- DC Series
---
&emsp;&emsp;[Vulnhub](https://www.vulnhub.com/)之DC系列最后一台靶机[DC-9](http://www.five86.com/downloads/DC-9.zip)的Write-up。

<!-- more -->

## Step 1

&emsp;&emsp;老规矩还是先找靶机IP，然后做端口扫描：

![寻找靶机IP](/img/DC-9/DC-9-1.png)

![扫描开放端口](/img/DC-9/DC-9-2.png)

## Step 2

&emsp;&emsp;访问80端口的WEB界面：

![](/img/DC-9/DC-9-3.png)

&emsp;&emsp;最开始在search.php界面尝试了很久的SQL注入，甚至还用了Sqlmap，一无所获。后来经人提醒，在search之后的results.php界面存在注入漏洞，注入点为包的数据部分的search参数，所以sqlmap命令需要带上`--data="search=1"`参数。

![获取数据库名](/img/DC-9/DC-9-4.png)

![获取users数据库的表](/img/DC-9/DC-9-5.png)

![获取users数据库中的数据](/img/DC-9/DC-9-6.png)

![获取Staff数据库中的表](/img/DC-9/DC-9-7.png)

![在Staff数据库的Users表中找到admin](/img/DC-9/DC-9-8.png)

&emsp;&emsp;很明显admin的密码是md5加密的，通过解密得到admin用户的密码：transorbital1

![](/img/DC-9/DC-9-9.png)

## Step 3

&emsp;&emsp;通过admin账户登录系统：

![](/img/DC-9/DC-9-10.png)

&emsp;&emsp;发现多了一个页面Add Record：

![](/img/DC-9/DC-9-11.png)

&emsp;&emsp;在这个页面再次进行了多次的尝试依然一无所获。但是在Manage界面和Add Record界面下方都发现奇怪的`File does not exist`提示，大胆猜测存在本地文件包含漏洞，经过测试，果然在Manage界面找到这个漏洞：

![](/img/DC-9/DC-9-12.png)

&emsp;&emsp;但是这个漏洞需要结合其他的漏洞组合利用，所以到这一步再次停顿。

## Step 4

&emsp;&emsp;通过Internet Surfing了解到这里有个端口敲门服务（knockd），这是一种端口试探服务器工具。它侦听以太网或其他可用接口上的所有流量，等待特殊序列的端口命中(port-hit)。telnet或Putty等客户软件通过向服务器上的端口发送TCP或数据包来启动端口命中。

&emsp;&emsp;简单来说，就是需要知道ssh服务的自定义端口，然后依次发送数据包“敲门”，从而开启ssh服务。默认配置文件为`/etc/knockd.conf`，所以现在的思路就是通过上一步的本地文件包含漏洞，查看这个配置文件，获得自定义端口，然后依次敲门，开启靶机的算是服务。

![](/img/DC-9/DC-9-13.png)

&emsp;&emsp;接下来是依次敲门：

```Bash
namp -p 7469 172.16.83.135
namp -p 8475 172.16.83.135
namp -p 9842 172.16.83.135
```

![](/img/DC-9/DC-9-14.png)

&emsp;&emsp;敲门之后再次查看ssh端口，发现ssh服务已经开启（对比 Step 1 中的22端口扫描信息）。

## Step 5

&emsp;&emsp;ssh服务开启后下一步就该是寻找登录凭证了，将在数据库中找到的所有用户信息综合起来进行22端口的目录爆破。

&emsp;&emsp;找到下列可用的账户：

![](/img/DC-9/DC-9-15.png)

&emsp;&emsp;登录这三个账户仅在jamitor中发现一点有用的信息：

![](/img/DC-9/DC-9-16.png)

&emsp;&emsp;这是一堆密码，把这些密码加入到制作的密码字典中去，再进行ssh爆破，发现多出来一个新的账号：

![](/img/DC-9/DC-9-17.png)

&emsp;&emsp;用这个新的账号再次登录，也找不到什么有用的文件，但是当用`sudo -l`命令查看sudo权限可以执行什么命令时，发现一个奇怪的脚本：

![](/img/DC-9/DC-9-18.png)

&emsp;&emsp;找到这个脚本看一下具体是什么内容：

![](/img/DC-9/DC-9-19.png)

&emsp;&emsp;分析一下这个脚本发现通过这个程序可以以root权限合并文件内容，那么通过该脚本可以实现讲一个root权限的用户写入/etc/passwd中去。

## Step 6

&emsp;&emsp;首先利用openssl创建一个本地加密用户：`openssl -1 -salt admin1 admin1`

&emsp;&emsp;-1 的意思是使用md5加密算法，-salt意思是自动插入一个随机数作为文件内容加密，即加盐：

![](/img/DC-9/DC-9-20.png)

&emsp;&emsp;然后在靶机中运行test程序将刚才创建的用户写入到`/etc/passwd`中去，切换到新添加的用户即可：

![](/img/DC-9/DC-9-21.png)

&emsp;&emsp;然后切换到root用户文件夹，接可以找到flag文件：

![](/img/DC-9/DC-9-22.png)
