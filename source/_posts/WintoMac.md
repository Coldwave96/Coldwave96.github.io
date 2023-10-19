---
title: 将Hexo从Windows迁移到Mac上的踩坑教学
date: 2020-05-01 11:36:13
categories:
- Miscellaneous
- Sites
tags:
- Blogs
- Hexo
---
我终于换MAC了！！！我终于换MAC了！！！我终于换MAC了！！！

重要的事情说3遍，其实我已经换MAC半年了，但是依然很兴奋，毕竟这台电脑的价格比我之前所有电脑加起来都贵==。刚换的时候就想把hexo转移到MAC，拖到现在才搬家的理由只有一个：懒！！！

话不多说，开始踩坑。

<!-- more -->

## 安装git和node.js

`brew install git`

`brew install node`

至于brew，emm，baidu了解一下。

## 安装hexo

使用node.js来安装：

`npm install hexo-cli -g`

这是第一个坑，一开始我用的是`npm install hexo g`命令安装的hexo，虽然可以成功安装hexo，可是用的时候会出现`hexo: COMMAND NOT FOUND`问题。

## 初始化hexo

新建一个hexo目录，比如`/Blog`这种，然后`hexo init`

再用`hexo s`测试是否成功，打开`localhost:4000`查看本地hexo界面是否正常显示。

## 生成SSH Key

先查看本地的SSH Key：`cd ~/.ssh`

如果没有，生成一个SSH Key：`ssh-keygen -t rsa -C "example@example.com"`，最后那个是注册邮箱。

## 将SSH Key添加到Github

进入.ssh文件夹：`cd ~/.ssh`，然后打开里面的`id_rsa.pub`文件，里面的内容就是SSH key，复制全部内容；

网页打开github的设置：Settings -> SSH and GPG keys，点击绿色的按钮New SSH key，然后在输入框中输入刚才复制的内容；

保存后，github可能会向你的邮箱发送一个验证链接（如果有记得去邮箱验证，不然之后的hexo部署会一直不成功的）。

测试一下是否成功：`ssh git@github.com`

看到下面的即成功：

```Bash
PTY allocation request failed on channel 0
Hi gjincai! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

## 文件配置转移

windows下的博客根目录hexo，复制该目录下的：`_config.yml`, `scaffolds`, `source`, `themes`；

直接覆盖替换mac下的博客根目录hexo相同的文件和文件夹。

## 坑

理论上这样就完成了，但是实际上还会有问题。

这是第二个坑，这时`hexo g && hexo d`依然不成功，是因为缺少了模块，需要执行命令`npm install hexo-deployer-git --save`

你以为这样就完成了么，不，下面是第三个坑。

hexo重新部署之后，博客界面显示`extends includes/layout.pug block content include includes/recent-posts.pug include includes/partial`

解决方案：

* 执行命令`npm install --save hexo-renderer-jade hexo-generator-feed hexo-generator-sitemap hexo-browsersync hexo-generator-archive`

* 清除缓存`hexo clean`

* 生成静态文件`hexo g`

这样就大功告成，再次`hexo d`部署就可以啦。