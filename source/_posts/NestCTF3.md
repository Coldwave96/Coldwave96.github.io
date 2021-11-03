---
title: CTF答题夺旗赛（第三季）WriteUPs
date: 2019-12-02 20:14:08
categories:
- WriteUPs
- CTF
tags:
- NEST
---
&emsp;&emsp;前几天参加了由[网络内生安全试验场](https://nest.ichunqiu.com/)（Network Endogens Security Testbed，NEST）举办的CTF答题夺旗赛（第三季）。题目整体比较简单，简单的写一下WriteUPs。

<!-- more -->

## weak

&emsp;&emsp;打开后就是一个某公司网站:

![](/img/NestCTF3/NestCTF3-1.png)

&emsp;&emsp;只有一个管理平台可以点击，之后进入用户管理平台：

![](/img/NestCTF3/NestCTF3-2.png)

&emsp;&emsp;点击跳转到测试页发现一段源码如下图所示：

![](/img/NestCTF3/NestCTF3-3.png)

&emsp;&emsp;看过源码再结合题目名字weak，很显然就是一个php弱等于的问题：php中有两种比较的符号 == 与 === ，=== 在进行比较的时候，会先判断两种字符串的类型是否相等，再比较== 在进行比较的时候，会先将字符串类型转化成相同，再比较, 如果遇到了0e\d+这种字符串，就会将这种字符串解析为科学计数法。"0e132456789" == "0e7124511451155"中 2 个数的值都是 0 因而就相等了。如果不满足0e\d+这种模式就不会相等。

&emsp;&emsp;首先是判断是否通过POST传入username和password两个数据，如果传入了就进行下面的判断，看到最下面，如果logined为TRUE就输出flag，所以需要使前两个if里面的结果为False，第三个if里的结果为True。

&emsp;&emsp;第一个判断中的ctype_alpha函数是做纯字符检测 ， 如果在当前语言环境中传入参数里的每个字符都是一个字母，那么就返回TRUE，反之则返回FALSE，然后前面有一个!所以username需要传入 的全部是字母。

&emsp;&emsp;第二个判断中的is_numeric函数是检测变量是否为数字或数字字符串，如果是数字和数字字符串则返回 TRUE，否则返回 FALSE。 所以password需要传入纯数字。

&emsp;&emsp;第三个判断的意思是username和password两个的md5值相同，但是两者本身不同，用到的就是刚开始说的php弱等于，下面是一些收集到的md5加密后为0e开头的字符串:

&emsp;&emsp;纯字母：

| 明文 | 密文 |
|:-----:|:-----:|
| UYXFLOI | 0e552539585246568817348686838809 |
| PJNPDWY | 0e291529052894702774557631701704 |
| DYAXWCA | 0e424759758842488633464374063001 |

&emsp;&emsp;纯数字：

| 明文 | 密文 |
|:-----:|:-----:|
| 571579406  | 0e972379832854295224118025748221 |
| 3465814713 | 0e258631645650999664521705537122 |
| 5432453531 | 0e512318699085881630861890526097 |

&emsp;&emsp;通过上面的表，用户名输入UYXFLOI，密码输入571579406，就可以获得flag。

![](/img/NestCTF3/NestCTF3-4.png)

## md5_brute

&emsp;&emsp;打开之后一步一步跟着文本的指导，就可以得到flag。

![](/img/NestCTF3/NestCTF3-5.png)

## help

&emsp;&emsp;打开之后是和weak差不多的某公司网站，只有页面上的“帮助”点击之后有反应，一看地址栏是http://120.55.43.255:17325/?file=help.php， 很轻易的就能联想到file参数可能存在文件包含漏洞，经过测试果然存在本地文件包含。

&emsp;&emsp;在首页查看源代码，在最底部找到这样的提示：

![](/img/NestCTF3/NestCTF3-6.png)

&emsp;&emsp;所以提交file=../../flag即可获得flag。

![](/img/NestCTF3/NestCTF3-7.png)

## word

&emsp;&emsp;这可能是最有（wu）趣（liao）的题目了，一个加密的word文件，密码在题目中告诉你了，然后就没有然后了。

&emsp;&emsp;最开始我还以为给出的flag是某一种算法加密后的，后来……

&emsp;&emsp;好吧，出题人你赢了==

## search

&emsp;&emsp;还是熟悉的某公司网站，搜索框随便输个数字，得到这样一个页面：

![](/img/NestCTF3/NestCTF3-8.png)

&emsp;&emsp;再看一下网址：http://120.55.43.255:11777/search.php?id=1，id参数必然存在注入漏洞。下面的方法随便手工注入或者sqlmap都可以轻松的搞定。

## encrypt

&emsp;&emsp;打开之后是这样一段数字和字母：

```code
69725f765f61797d74797465667321275f6f5f6c796573655f746121615f61736867655376736f697b417965796c73457321
```

&emsp;&emsp;这是一段hex编码的字符串，揭开之后是：

```code
ir_v_ay}tytefs!'_o_lyese_ta!a_ashgeSvsoi{AyeylsEs!
```

&emsp;&emsp;显然已经出来了，只剩最后一步。这是栅栏密码，经过尝试，发现为7个一组，解码之后可以得到flag。

## 唱跳rap篮球

&emsp;&emsp;emmmmm，最无语的题。打开就是个登陆界面，查看源码发现最后由这样的提示：

![](/img/NestCTF3/NestCTF3-9.png)

&emsp;&emsp;所以怎么办呢，caixukun/19980802登陆就可以获得flag：

![](/img/NestCTF3/NestCTF3-10.png)

&emsp;&emsp;emmmmm，除了无语还是无语……

## code

&emsp;&emsp;又是编码题，打开之后又是一堆奇怪的数字：

```code
9a9a9a6a9aa9656699a699a566995956996a996aa6a965aa9a6aa596a699665a9aa699655a696569655a9a9a9a595a6965569a59665566955a6965a9596a99aa9a9566a699aa9a969969669aa6969a9559596669
```

&emsp;&emsp;最奇怪的是只有5，6，9，a这4种字符，找了很久，这居然是曼彻斯特编码，还是差分曼彻斯特编码：5:0101；6:0110；9:1001；a：1010。然后转换为16进制，再转换成字符串即可获得flag。

&emsp;&emsp;具体的可以看[这里](http://antcave.cn/archives/400)。

## upload

&emsp;&emsp;打开之后是一个简单的上传界面，只能上传png格式的文件，且验证过程是在后台进行，所以无法绕过。查看源码，发现一个有趣的首页图片地址：(这题刚开始没多久就已经被师傅们玩坏了，没有截图，只能自己想象了)

&emsp;&emsp;http://120.55.43.255:11881/include.php?file=dXBsb2Fk

&emsp;&emsp;file提交的参数经过base64解码之后结果是upload。有趣，上传的图片文件名会被base64编码。但是当我们通过file参数提交上传的图片名base64编码后的字符串之后，可以访问图片，且图片文件被解析成脚本。

&emsp;&emsp;分析清楚之后，那接下来就很简单了，创建一个一句话的图片马，上传之后用蚁剑去连接，flag文件就在根目录下。

## 后记

&emsp;&emsp;关于PWN和逆向的题，别问，问就是简（bu）单（hui），不（xu）值（yao）一（xue）提（xi）。
