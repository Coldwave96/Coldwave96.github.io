---
title: Hexo + Github搭建免费个人博客
date: 2019-11-06 11:25:28
categories: 
- Miscellaneous
- Sites
tags:
- Blogs
- Hexo
---
&#160; &#160; &#160; &#160;[Github](https://github.com)是收到广大程序猿追捧的网站，各种源码资源应有尽有。[Hexo](https://hexo.io)是一款轻量级的Blog框架，安装和配置都很简单。两者结合即可搭建属于自己的免费博客，省时省力，既不用花钱买VPS，也不用花时间维护，这是属于白嫖党的胜利。

<!-- more -->

## Step 1：创建Github账户

&#160; &#160; &#160; &#160;创建[Github](https://github.com)账户这一步比较简单，就不赘述了。

## Step 2：创建Github Pages

### 2.1 创建仓库

&#160; &#160; &#160; &#160;首先新建一仓库，注意仓库的名字一定要为[Owner].github.io，这样才可以通过https://[Owner].github.io访问我们创建的Github Pages。

![](/img/CreateBlogs/CreateBlogs1.png)

![](/img/CreateBlogs/CreateBlogs2.png)

### 2.2 任意选择一个主题

&#160; &#160; &#160; &#160;对新建的仓库选择settings，然后将设置页面拉倒最底下，找到GitHub Pages部分，任意选择一个主题即可。

![](/img/CreateBlogs/CreateBlogs3.png)

![](/img/CreateBlogs/CreateBlogs4.png)

![](/img/CreateBlogs/CreateBlogs5.png)

![](/img/CreateBlogs/CreateBlogs6.png)

&#160; &#160; &#160; &#160;这样就完成了Github Pages的设置，访问我们的博客主页https://[Owner].github.io，就可以看到暂时搭建好的主页模版。

## Step 3：安装Git

&#160; &#160; &#160; &#160;前往[Git官网](https://git-scm.com/)下载安装即可，安装过程中所有选项均默认即可。

## Step 4：安装Node.js

&#160; &#160; &#160; &#160;前往[Node.js官网](https://nodejs.org/en/)下载安装即可，安装过程中的所有选项也均默认即可。

## Step 5：安装Hexo

&#160; &#160; &#160; &#160;下面是最关键的一步了，就是安装Hexo。创建一个用来存放Hexo组件的目录，例如D:\Blogs，进入该目录，右键选择Git Bash here。

![](/img/CreateBlogs/CreateBlogs7.png)

&#160; &#160; &#160; &#160;然后使用npm安装hexo客户端：

```bash
$ npm install hexo-cli -g
```

&#160; &#160; &#160; &#160;下载好hexo后，进行初始化：

```bash
$ hexo init
```

&#160; &#160; &#160; &#160;查看所安装的hexo版本：

```bash
$ hexo -v
```

&#160; &#160; &#160; &#160;查看帮助说明文档：

```bash
$ hexo h
```

&#160; &#160; &#160; &#160;打开本地博客目录下的_config.yml文件，最下面找到Deployment模块，将标记的部分替换成自己的[Owner]即可：

![](/img/CreateBlogs/CreateBlogs8.png)

&#160; &#160; &#160; &#160;使用hexo s在本地4000端口开启服务，浏览器访问http://127.0.0.1:4000 即可看到我们的博客首页：

```bash
$ hexo s
```
&#160; &#160; &#160; &#160;这样个人博客就算搭建成功了，之后的工作就是美化自己的博客，发布blogs了。当然所有要发布的blogs需要用markdown语法去写，然后保存为.md格式的文件，放到之前你创建的文件夹下的/source/_posts/m目录下。

&#160; &#160; &#160; &#160;本地写好博文，用hexo s命令在本地查看效果修改，终稿使用hexo g && hexo d命令发布到你的Github Pages上去，过程中可能会要求你输入自己的Github帐号和密码。

## Options

### Options 1：主题

&#160; &#160; &#160; &#160;按照上述步骤搭建好的个人博客，主题应该是hexo自带的landscape风格，如果不喜欢的话，[这里](https://hexo.io/themes/)有很多不同风格的主题，大家可以选择自己喜欢的。

### Options 2：配置

&#160; &#160; &#160; &#160;包括hexo的配置以及自己更换的主题的配置，都是通过修改_config.yml文件实现。关于hexo的配置，[这个页面](https://hexo.io/docs/configuration)也许对你有用。至于不同的主题的配置，则需要自己去找一下喽。

## Problems

### Problem 1：找不到Git

&#160; &#160; &#160; &#160;在执行hexo d命令时可能会出现这个错误，解决方法：

```bash
npm install hexo-deployer-git --save
```

### Problem 2：无法自动检测邮箱

&#160; &#160; &#160; &#160;同样在执行hexo g && hexo d命令时可能会出现该问题，解决方法会在报错信息中给出，只需执行相应命令即可。
