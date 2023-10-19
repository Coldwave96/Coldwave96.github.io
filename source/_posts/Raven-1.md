---
title: Vulnhub - Raven 1 の Write-Up
date: 2020-05-25 16:02:08
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- Raven Series
---
## 前言

[Vulnhub](https://www.vulnhub.com/)之Raven系列靶机第一台[Raven 1](https://www.vulnhub.com/entry/raven-1,256/)的Write-up。

<!-- more -->

## Step 1

寻找靶机地址：

![](/img/Raven-1/Raven-1-1.png)

扫描开放的端口和服务：

![](/img/Raven-1/Raven-1-2.png)

首先看下80端口的服务，打开后是Raven Security公司的网站，浏览下网站，在SERVICE界面的源码中找到flag1：

![](/img/Raven-1/Raven-1-3.png)

## Step 2

在BLOG界面，发现是一个wordpress搭建的站点，后台数据库为MySQL：

![](/img/Raven-1/Raven-1-4.png)

根据经验这里肯定是有问题的，用wpscan扫一下看看能发现什么：

![](/img/Raven-1/Raven-1-5.png)

然而并没有得到什么有用的信息，不甘心，再尝试爆破一下用户信息：

![](/img/Raven-1/Raven-1-6.png)

找到两个用户steven和michael，尝试爆破这两个账户的密码无果。

## Step 3

在wordpress这里尝试了很久也没有任何收获，只能转向22端口。

尝试下steven和michael两个用户爆破22端口：

![](/img/Raven-1/Raven-1-7.png)

找到一个，接下来ssh登录一下，找找有什么信息。

![](/img/Raven-1/Raven-1-8.png)

登录后提示我们有新的邮件，去找一下这个邮件在哪里：

![](/img/Raven-1/Raven-1-9.png)

然后看一下michael这封邮件内容，有一个比较有趣的片段：

![](/img/Raven-1/Raven-1-10.png)

但是并不明白这有什么用。同时在/var/www目录下找到flag2.txt文件：

![](/img/Raven-1/Raven-1-11.png)

## Step 4

在查看wordpress网站的配置文件时，找到数据库的账户和密码，之前Wappalyzer也说明网站后台是MySQL数据库：

![](/img/Raven-1/Raven-1-12.png)

连接MySQL数据库：

![](/img/Raven-1/Raven-1-13.png)

![](/img/Raven-1/Raven-1-14.png)

wp_users表里应该是用户的账户密码，看一下有什么：

![](/img/Raven-1/Raven-1-15.png)

果然是Steven和Michael两个用户，Michael用户的密码查不出来，Steven账户密码解密结果：

![](/img/Raven-1/Raven-1-16.png)

再依次看一下其他的表，在wp_posts表里发现flag3和flag4：

![](/img/Raven-1/Raven-1-17.png)

## Step 5

到这所有的flag我们都已经拿到了，但是还希望可以获得root权限。

虽然在数据库里获得的是wordpress后台账户的密码，但还是尝试一下ssh是否有用：

![](/img/Raven-1/Raven-1-18.png)

幸运地登录了steven账户，看一下sudo有什么权限：

![](/img/Raven-1/Raven-1-19.png)

那么就可以通过python提权了，具体可看之前DC系列靶机的writeups中python建立交互式终端：

![](/img/Raven-1/Raven-1-20.png)

提权成功，获取root权限后拿到一切（原来flag4应该这样拿==）：

![](/img/Raven-1/Raven-1-21.png)


## P.S.

靶机介绍里也说了这台靶机提权的方法有至少两种，后来看其他Writeup发现还可以通过MySQL的udf提权实现。关于udf提权在[JarvisOJ Web RE？题](https://coldwave96.github.io/2020/05/01/JarvisOJ-WEB-2/#RE)里有涉及到。

这台靶机udf提权首先找一下payload：

![](/img/Raven-1/Raven-1-22.png)

然后在本地编译好1518.c文件：

![](/img/Raven-1/Raven-1-23.png)

将编译好的1518.so文件上传到靶机上：

![](/img/Raven-1/Raven-1-24.png)

然后回到michael账户连接数据库开始提权：

![](/img/Raven-1/Raven-1-25.png)

![](/img/Raven-1/Raven-1-26.png)

![](/img/Raven-1/Raven-1-27.png)
