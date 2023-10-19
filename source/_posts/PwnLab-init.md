---
title: Vulnhub - PwnLab：init の Write-Up
date: 2020-10-22 15:01:36
categories:
- WriteUPs
- Vulnhub
tags:
- Web
- Reverse
---
## Description

`Vulnhub`靶机`PwnLab：init`的练习记录。整体难度不高，最终目的是拿到`/root/flag.txt`文件。

<!-- more -->

## Gathering Information

首先找下靶机地址：

![](/img/PwnLab_init/PwnLab1.png)

扫描一下靶机的开放端口和服务：

![](/img/PwnLab_init/PwnLab2.png)

## Game Start

先看下80端口的服务：

![](/img/PwnLab_init/PwnLab3.png)

Login界面是一个看起来很简陋的登陆界面：

![](/img/PwnLab_init/PwnLab4.png)

尝试对登录接口爆破和SQL注入均无结果。

但是针对界面的URL：`http://172.16.83.142/?page=login和http://172.16.83.142/?page=upload`的形式，猜测可能存在文件包含的问题。

结合`nikto`给出的信息：

![](/img/PwnLab_init/PwnLab5.png)

`/config.php`文件存在可能泄露数据库的信息，但是直接访问是没有数据的。

接下来就尝试读取`/config.php`文件的内容，详细内容可参考这篇文章[php伪协议实现命令执行的七种姿势](https://www.freebuf.com/column/148886.html)。最终通过`php://filter`实现内容读取，这个技巧也是CTF中经常用到的。

最终payload是这样的：`http://172.16.83.142/?page=php://filter/read=convert.base64-encode/resource=config`（注意不用加.php后缀）

![](/img/PwnLab_init/PwnLab6.png)

base64解码出来之后的结果中果然含有数据库的信息：

![](/img/PwnLab_init/PwnLab7.png)

同样的方法我们可以获得`login.php`：

```PHP
<?php
session_start();
require("config.php");
$mysqli = new mysqli($server, $username, $password, $database);

if (isset($_POST['user']) and isset($_POST['pass']))
{
	$luser = $_POST['user'];
	$lpass = base64_encode($_POST['pass']);

	$stmt = $mysqli->prepare("SELECT * FROM users WHERE user=? AND pass=?");
	$stmt->bind_param('ss', $luser, $lpass);

	$stmt->execute();
	$stmt->store_Result();

	if ($stmt->num_rows == 1)
	{
		$_SESSION['user'] = $luser;
		header('Location: ?page=upload');
	}
	else
	{
		echo "Login failed.";
	}
}
else
{
	?>
	<form action="" method="POST">
	<label>Username: </label><input id="user" type="test" name="user"><br />
	<label>Password: </label><input id="pass" type="password" name="pass"><br />
	<input type="submit" name="submit" value="Login">
	</form>
	<?php
}
```

以及`upload.php`:

```PHP
<?php
session_start();
if (!isset($_SESSION['user'])) { die('You must be log in.'); }
?>
<html>
	<body>
		<form action='' method='post' enctype='multipart/form-data'>
			<input type='file' name='file' id='file' />
			<input type='submit' name='submit' value='Upload'/>
		</form>
	</body>
</html>
<?php 
if(isset($_POST['submit'])) {
	if ($_FILES['file']['error'] <= 0) {
		$filename  = $_FILES['file']['name'];
		$filetype  = $_FILES['file']['type'];
		$uploaddir = 'upload/';
		$file_ext  = strrchr($filename, '.');
		$imageinfo = getimagesize($_FILES['file']['tmp_name']);
		$whitelist = array(".jpg",".jpeg",".gif",".png"); 

		if (!(in_array($file_ext, $whitelist))) {
			die('Not allowed extension, please upload images only.');
		}

		if(strpos($filetype,'image') === false) {
			die('Error 001');
		}

		if($imageinfo['mime'] != 'image/gif' && $imageinfo['mime'] != 'image/jpeg' && $imageinfo['mime'] != 'image/jpg'&& $imageinfo['mime'] != 'image/png') {
			die('Error 002');
		}

		if(substr_count($filetype, '/')>1){
			die('Error 003');
		}

		$uploadfile = $uploaddir . md5(basename($_FILES['file']['name'])).$file_ext;

		if (move_uploaded_file($_FILES['file']['tmp_name'], $uploadfile)) {
			echo "<img src=\"".$uploadfile."\"><br />";
		} else {
			die('Error 4');
		}
	}
}

?>
```

通过泄露的数据库账号密码可以通过root远程登录mysql数据库：

![](/img/PwnLab_init/PwnLab8.png)

根据提示在Users表中找到了3个账号。

![](/img/PwnLab_init/PwnLab9.png)

密码很明显是base64编码的，解码之后得到3对账户和密码：

* kent/JWzXuBJJNy

* mike/SIfdsTEn6I

* kane/iSv5Ym2GRo

登录系统之后直接跳转到upload界面，允许上传文件。但是通过上面我们down下来的`upload.php`源码来看，上传文件是后端的白名单验证后缀，而且上传上去之后会改文件名再添加后缀。但是只是检查了`imageinfo`，那可以伪造一个文件头加后缀就可以了。

就在kali自带的php webshell里挑一个`php-reverse-shell.php`改一下就可以。

![](/img/PwnLab_init/PwnLab10.png)

先加个文件头：

![](/img/PwnLab_init/PwnLab11.png)

再把反连IP和端口改成自己的：

![](/img/PwnLab_init/PwnLab12.png)

最后文件名加上`.gif`的后缀，就可以成功上传：

![](/img/PwnLab_init/PwnLab13.png)

## Hint

但是又出现了新的问题，我们没法访问这个图片……

在网上找了一些提示之后才想起来，我们下载了`config.php`，`login.php`，`upload.php`3个源码文件，但是还有主页的源码没看。再次利用php伪协议读取主页源代码：

```HTML
<?php
//Multilingual. Not implemented yet.
//setcookie("lang","en.lang.php");
if (isset($_COOKIE['lang']))
{
	include("lang/".$_COOKIE['lang']);
}
// Not implemented yet.
?>
<html>
<head>
<title>PwnLab Intranet Image Hosting</title>
</head>
<body>
<center>
<img src="images/pwnlab.png"><br />
[ <a href="/">Home</a> ] [ <a href="?page=login">Login</a> ] [ <a href="?page=upload">Upload</a> ]
<hr/><br/>
<?php
	if (isset($_GET['page']))
	{
		include($_GET['page'].".php");
	}
	else
	{
		echo "Use this server to upload and share image files inside the intranet";
	}
?>
</center>
</body>
</html>
```

可以看到cookie里有个有趣的参数`lang`可以实现访问任意文件：

![](/img/PwnLab_init/PwnLab14.png)

于是利用这个漏洞触发webshell反弹shell:

![](/img/PwnLab_init/PwnLab15.png)

可以看到利用Burp发包之后kali就接收到里反弹的shell。通过下面的命令即可转换成交互式shell。

```Bash
python -c '__import__("pty").spawn("/bin/bash")’
```

## Exploring

现在我们已经拿到目标机器的shell，下一步就是寻找有价值的信息。在`home`下发现4个用户：

![](/img/PwnLab_init/PwnLab16.png)

尝试切换到这几个用户，`kent`下什么也没有：

![](/img/PwnLab_init/PwnLab17.png)

`mike`的密码不对：

![](/img/PwnLab_init/PwnLab18.png)

在`kane`目录下找到一个奇怪的文件`msgmike`：

![](/img/PwnLab_init/PwnLab19.png)

是个可执行文件且存在`setuid`，`setgid`。执行发现调用了`cat`命令：

![](/img/PwnLab_init/PwnLab20.png)

接下来将`cat`命令换成`bash`：

![](/img/PwnLab_init/PwnLab21.png)

通过`export PATH=.:$PATH`命令修改一下系统路径PATH，再次运行`msgmike`就可以得到`mike`的bash：

![](/img/PwnLab_init/PwnLab22.png)

回到`mike`的目录下，有个新的`msg2root`的程序，根据名字感觉是提权到`root`权限的东西：

![](/img/PwnLab_init/PwnLab23.png)

但是看起来就是简单的将输入变成输出：

![](/img/PwnLab_init/PwnLab24.png)

联想到`PwnLab`的名字，通过`scp`命令将这个程序传回本地，逆向分析一下：

![](/img/PwnLab_init/PwnLab25.png)

`msg2root`逆向后的主函数伪代码如下：

![](/img/PwnLab_init/PwnLab26.png)

其实是个很简单的程序，将输入写入到`/root/message.txt`中，然后`system`执行。那通过`;`截断就可以绕过了。

![](/img/PwnLab_init/PwnLab27.png)

## Finished

可以看到我们已经得到了`root`的权限，最后就是去根目录下读`flag.txt`文件了：

![](/img/PwnLab_init/PwnLab28.png)

完结，撒花～～
