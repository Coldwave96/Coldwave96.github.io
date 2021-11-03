---
title: Vulnhub - DC-2 の Write-Up
date: 2019-11-19 15:24:59
categories: 
- WriteUPs
- Vulnhub
tags: 
- Web
- DC Series
---
## 前言

&emsp;&emsp;[Vulnhub](https://www.vulnhub.com/)之DC系列靶机第二台[DC-2](http://www.five86.com/downloads/DC-2.zip)的Write-up。

<!-- more -->

## Step 1

&emsp;&emsp;老套路，第一步首先寻找靶机IP，`arp-scan -l`寻找“陌生”机器。

![](/img/DC-2/DC-2-1.png)

&emsp;&emsp;然后扫描一下靶机开放端口，很奇怪，常用端口只开放了80（伏笔）。

![](/img/DC-2/DC-2-2.png)

## Step 2

&emsp;&emsp;查看80端口，发现网站打不开。根据Nmap扫描加过的提示，发现需要在hosts文件里添加`172.16.12.128 dc-2`。成功访问网站之后，发现是Wordpress cms，并在flag界面找到第一个flag。

![](/img/DC-2/DC-2-3.png)

&emsp;&emsp;根据第一个flag的提示，可能需要爆破网站后台登录名和密码。通过wpscan工具扫描网站并列举网站用户，找到admin、tom、jerry这3个用户。

![](/img/DC-2/DC-2-4.png)

![](/img/DC-2/DC-2-5.png)

## Step 3

&emsp;&emsp;第一个flag的提示同时也告诉我们，密码可能在普通的字典中没有。更是提示我们利用cewl这个工具去生成本次爆破需要的字典。

&emsp;&emsp;使用`cewl http://dc-2/ > dc2.txt`命令生成密码字典，然后用wpscan爆破admin、tom、jerry用户的密码，最终得到tom和jerry用户密码。

![](/img/DC-2/DC-2-6.png)

&emsp;&emsp;尝试两个用户登录，果然如第一个flag提示那样，只有jerry用户可以登录后台，并在后台中发现flag2。

![](/img/DC-2/DC-2-7.png)

## Step 4

&emsp;&emsp;仔细阅读flag2的提示，原来这个靶机还有一个入口。那这个入口在哪呢？Step 1的伏笔就在这里，再对靶机进行全端口扫描，果然发现了第二个入口，7744端口的ssh服务。

![](/img/DC-2/DC-2-8.png)

&emsp;&emsp;这时自然而然想到爆破出来的那两个账户是不是依然有用，特别是没能登录进后台的tom账户。尝试了一下，果然这次jerry账户无法登陆，tom账户可以成功ssh连接上靶机。

![](/img/DC-2/DC-2-9.png)

## Step 5

&emsp;&emsp;tom账户登陆之后发现几乎所有命令都不能使用，是一个严重受限制的rbash。多次尝试发现vi居然可以用，通过vi看到了第三个flag文件。

![](/img/DC-2/DC-2-10.png)

![](/img/DC-2/DC-2-11.png)

## Step 6

&emsp;&emsp;flag3好像又提示我们要切换到jerry用户，那只能再次尝试，su，sudo等等等等，全部没用。只能上网寻找如何跳出rbash限制，功夫不负有心人。踩了很多很多坑之后找到了有用的[帮助](https://www.cnblogs.com/xiaoxiaoleo/p/8450379.html)。

![](/img/DC-2/DC-2-12.png)

&emsp;&emsp;切换到jerry账户，然后找到了flag4文件。

![](/img/DC-2/DC-2-13.png)

## Step 7

&emsp;&emsp;看到flag4文件的叙述，果然最后还是要到root文件夹下寻找最后一个flag。可是如何切换到root用户呢，老套路又来了，还记得DC-1靶机时说道的[SUID提权](https://coldwave96.github.io/2019/11/15/suid/)么。

&emsp;&emsp;其实flag4文件最后已经给我们提示了，嘴上很无情，身体还是很诚实的嘛。

&emsp;&emsp;对的，就是通过git提权。

![](/img/DC-2/DC-2-14.png)

&emsp;&emsp;拿到root权限之后，在/root文件夹下找到最后一个flag。

![](/img/DC-2/DC-2-15.png)
