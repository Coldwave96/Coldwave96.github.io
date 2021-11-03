---
title: PocLibrary - 界面版POC/EXP仓库
date: 2020-08-24 14:22:23
categories:
- Tools
- Program
- Python
tags:
- Github
---
## 项目起源

&emsp;&emsp;主要在于自己的强迫症。

&emsp;&emsp;在做渗透测试或者漏洞验证工作，以及平时玩靶机的时候收集了大量的POC/EXP的脚本，包括在Github上fork了各路大神写的脚本，导致本地文件夹和Github仓库混乱，找起来不是很方便。正好看到了[@zhzyker的exphub项目](https://github.com/zhzyker/exphub)，于是借着这个机会把POC/EXP脚本规整一下，做一个界面化的工具出来，就有了[PocLibrary](https://github.com/Coldwave96/PocLibrary)项目。

<!-- more -->

## UI搭建

&emsp;&emsp;很久很久之前用C#做过GUI的开发，长时间没用C#，现在已经忘的差不多了。只能去了解一下Python的GUI开发。

&emsp;&emsp;Python下的开发GUI的库大体上有PyQt，wxPython和Tkinter。由于这次的界面设计的也比较简单，3种框架差别不大。刚好wxPython有一个类似C# MFC的工具[wxFormBuilder](https://github.com/wxFormBuilder/wxFormBuilder)，于是最终选择了wxPython框架开发GUI界面。

&emsp;&emsp;首先设计一个选择POC/EXP模块的主界面：

![](/img/PocLibrary/PocLibrary1.png)

&emsp;&emsp;然后再设计一个显示某个POC/EXP脚本信息的子界面：

![](/img/PocLibrary/PocLibrary2.png)

&emsp;&emsp;考虑到错误处理，于是再创建一个错误信息告警界面：

![](/img/PocLibrary/PocLibrary3.png)

&emsp;&emsp;这时wxFormBuilder会帮我们自动生成好界面的代码：

![](/img/PocLibrary/PocLibrary4.png)

&emsp;&emsp;可以看到有很多种语言的版本，我们选择Python版的。

## 功能实现

&emsp;&emsp;界面搭好之后就是按照界面上的预想功能去实现。在主界面上有一个下拉的List，可以选择POC/EXP模块。

![](/img/PocLibrary/PocLibrary5.png)

&emsp;&emsp;图中展示了1.0版本中的模块，目前总共有14个模块，具体参见[项目主页](https://github.com/Coldwave96/PocLibrary)。

&emsp;&emsp;选好模块之后，点击确定Button会触发Bind的事件，根据提交的模块的值去定制显示信息的子界面。在子界面中继续选择想要查看的POC/EXP脚本，点击确定Button显示POC/EXP信息、POC/EXP脚本的用法以及对应的漏洞信息。

![](/img/PocLibrary/PocLibrary6.png)

&emsp;&emsp;并提供了Copy功能复制POC/EXP脚本内容，但是目前有些脚本因为Bug无法复制，只能手动复制。

&emsp;&emsp;没有做自动执行的功能，主要是考虑到POC/EXP都是从各个地方收集而来，执行的命令包括执行时需要的环境都不一样，很难做到统一，所以目前就仅仅实现了简单的Copy供用户复制出来自行配环境运行。

## 总结

&emsp;&emsp;更多信息见项目主页。
