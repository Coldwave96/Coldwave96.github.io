---
title: 开源组件实现 Software Defined Perimeter
date: 2021-06-08 11:03:22
categories: 
- Security Framework
- Zero Trust
tags:
- Zero Trust
- SDP
---
&emsp;&emsp;零信任安全是当下一个比较火热的话题，很多厂家都在尝试将其落地，整合到企业安全框架之中，实现产品化。零信任安全其中一种比较可行的实现方案是通过SDP(Software Defined Perimeter)，本文尝试通过现有的开源组件实现SDP。

<!-- more -->

## 参考模型
&emsp;&emsp;本次尝试主要参考以下模型：

* [Google's BeyondCorp](https://www.beyondcorp.com)

* [Cloud Security Alliance model of Software Defined Perimeter](https://cloudsecurityalliance.org/group/software-defined-perimeter/#_overview)

## 工具列表
<a href="http://www.cipherdyne.org/">fwknop</a> - Used to allow the SDP server to remain completely hidden from unauthorized use.  With this tool, the gateway server can be configured with 0 inbound port access.  The net result is that the gateway server is more hardened against port scanning, DDoS attacks, etc.  This component will be optional as the client component is not readily available on all major platforms (ie. iPhone).  This project is definitely worth a look for anyone looking to contribute to a really awesome open source project!

<a href="http://www.squid-cache.org/">Squid</a> - Used to provide authorization to upstream resources.  Squid is being used because of it's ability to use external authentication helpers and assign access based on group memberships from either a common database, or LDAP server.  Squid also gives us the granularity to apply rules based on destination host, URI, port or a combination.

## 测试拓扑

![测试拓扑图](/img/SDP/SDP1.png)

## 方案简述

&emsp;&emsp;网络边界部署边界服务器，在边界服务器上安装Squid反向代理内网Web服务。同时在边界服务器上安装并开启fwknop-server，在客户端上安装fwknop-client，通过配置实现单包认证访问。

## 测试步骤

### Step 1

&emsp;&emsp;网络拓扑比较简单，就不赘述搭建过程，进入正题。在内网Web服务器上搭建HTTP网站，关于如何在Winserver 2008 R2上搭建网站也很简单就直接跳过。现在我们搭建好的网站是这样的：

![](/img/SDP/SDP2.png)

### Step 2

&emsp;&emsp;在边界服务器上安装Squid，由于本次边界服务器是Ubuntu系统，所以输入`sudo apt isntall squid`即可。然后通过`sudo vim /etc/squid/squid.conf`命令修改squid的配置文件。

```
#http_port 3128
http_port 10.0.0.11:80 accel vhost vport
cache_peer 192.168.88.10 parent 80 0 no-query no_digest originserver
```

&emsp;&emsp;修改完成后通过`sudo systemctl restart squid`命令重启squid，这时候访问`http://10.0.0.11`就可以看到Step 1中搭建的网站。至此，HTTP反向代理就完成了。

### Step 3

&emsp;&emsp;本次测试选用的客户端是Ubuntu，直接通过`sudo apt install fwknop-client`即可安装fwknop的客户端，本次测试安装的是2.6.9版本。如果是Windows端则要去[fwknop官网](http://www.cipherdyne.org/)去下载源代码自行编译。

&emsp;&emsp;安装好fwknop-client之后执行`sudo fwknop -A tcp/80 -a 10.0.0.14 -D 10.0.0.11 --key-gen --use-hmac --save-rc-stanza`生成单包认证的Key。命令执行完之后会生成一个`.fwknoprc`文件，同时会告知文件位置。

```
-A tcp/80           请求服务端打开的端口及其协议；
-a 10.0.0.14        客户端的IP地址；
-D 10.0.0.11        服务端的IP地址；
-key-gen            生成一个加密密钥；
--use-hmac          采用hmac加密认证方式；
--save-rc-stanza    保存以上参数的执行结果。
```

&emsp;&emsp;通过`sudo grep KEY /home/User/.fwknoprc`命令即可获取Key。

![](/img/SDP/SDP3.png)

### Step 4

&emsp;&emsp;在边界服务器上通过`sudo apt install fwknop-server`安装fwknop服务端。然后输入`sudo vim /etc/fwknop/access.conf`修改服务端参数，将Step 3中客户端的Key加入配置文件：

```
SOURCE                 ANY
REQUIRE_SOURCE_ADDRESS Y
OPEN_PORTS             tcp/80
KEY_BASE64             rvyA5SgenTMOagiBJJER4otC+6hdbOxXSZKW8ZN7Bsk=
HMAC_KEY_BASE64        MtCbW46/8PCOLk7BImbLhtwSuXbPmCIyecZvmuY5Nx8NQ1PLrrqgEEumgq7YjhDXS6cpwHX/wbZ6ZckoX6dI4A==
```

&emsp;&emsp;然后还需要修改`/etc/feknop/fwknopd.conf`文件，将监听的网卡修改为机器外网网卡：

```
# Define the ethernet interface on which we will sniff packets.
# Default if not set is eth0.  The '-i <intf>' command line option overrides
# the PCAP_INTF setting.
#
PCAP_INTF                   ens33;
```

&emsp;&emsp;最后通过iptables实现对80端口的隐藏：

```
sudo iptables -I INPUT 1 -i ens33  -p tcp --dport 80 -j DROP
sudo iptables -I INPUT 1 -i ens33 -p tcp --dport 80 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

&emsp;&emsp;运行`sudo fwknopd reload`命令重启fwknopd服务，可通过`sudo fwknopd -S`命令查看服务状态，通过`sudo fwknopd --fw-list-all`命令查看iptables规则。

&emsp;&emsp;这样边界服务器的设置就完成了。

### Step 5

&emsp;&emsp;此时所有地址都无法访问边界服务器的HTTP服务，nmap扫描的结果如下：

![](/img/SDP/SDP4.png)

&emsp;&emsp;而当我们在client上执行`sudo fwknop -n 10.0.0.11`命令发送单包认证后，会发现此时80端口已经对client开放（如果不做任何操作30s后会自动关闭，更多配置在服务端的fwknop配置文件中）：

![](/img/SDP/SDP5.png)

&emsp;&emsp;这个时候client就可以访问到搭建在内网的HTTP服务了。

## 后记

&emsp;&emsp;结合下面的代理工具我们可以实现一些自动化以及非Web服务的单包认证机制。

<a href="https://openvpn.net/index.php/open-source.html">OpenVPN</a> - Used to ensure a completely encrypted communication channel between personal devices (laptop, cell phone, etc) and the gateway server.  OpenVPN includes support on every major platform and is simple to adjust the configuration to the user's needs.  In our model, we are not using OpenVPN in the traditional sense of a VPN as the gateway server will not be configured to forward traffic directly to an upstream device.  OpenVPN also supports additional authentication plugins allowing things like two-factor authentication to become possible. OpenVPN also provides the awesome PKI tool easy-rsa. easy-rsa gives us the ability to provision and manage certificates for all of our components.

<a href="https://github.com/darkk/redsocks">Redsocks Proxy</a> - This tool will be used to forward non-web traffic through our Squid proxy.

&emsp;&emsp;以及更多的fwknop指导在[这里](http://www.cipherdyne.org/fwknop/docs/fwknop-tutorial.html)。