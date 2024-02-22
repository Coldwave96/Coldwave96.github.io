---
title: 基于深度神经网络的Webshell静态检测
date: 2024-02-22 14:40:08
categories:
- Program
- Python
tags:
- NLP
- Webshell
---
## 背景

Webshell作为黑客惯用的入侵工具，是以php、asp、jsp、perl、cgi、py等网页文件形式存在的一种命令执行环境。黑客在入侵一个网站服务器后，通常会将webshell后门文件与网站服务器WEB目录下正常网页文件混在一起，通过Web访问webshell后门进行文件上传下载、访问数据库、系统命令调用等各种高危操作，达到非法控制网站服务器的目的，具备威胁程度高，隐蔽性极强等特点。

本文尝试通过一个 TextCNN + 二分类网络合成的综合深度神经网络实现对于 Webshell 的静态检测。TextCNN 用于处理向量化后的词数组，二分类网络用于处理手动提取的数字化特征（文件的大小以及熵值等等）。

<!-- more -->

2019年曾经做过一个简单的 Webshell 检测系统。源代码通过 N-Gram 分割的方式，对分割后的字符结合 TF-IDF 技术建立词袋，然后通过简单的机器学习算法如 NB、SVM 等进行二分类。现在的合成网络在利用 TextCNN 深度神经网络自动提取特征的基础上，结合手动设计提取的数字化特征，如文件大小，文件熵等信息，实现综合分类网络，对于一句话木马以及混淆木马有着更好的检测能力。

## 数据集

原始数据集采集自 [Github](https://github.com)，下面是详细的仓库列表.

### 黑样本
1. [tennc/webshell](https://github.com/tennc/webshell)
2. [JohnTroony/php-webshells](https://github.com/JohnTroony/php-webshells)
3. [xl7dev/webshell](https://github.com/xl7dev/webshell)
4. [tutorial0/webshell](https://github.com/tutorial0/webshell)
5. [bartblaze/PHP-backdoors](https://github.com/bartblaze/PHP-backdoors)
6. [BlackArch/webshells](https://github.com/BlackArch/webshells)
7. [nikicat/web-malware-collection](https://github.com/nikicat/web-malware-collection)
8. [fuzzdb-project/fuzzdb](https://github.com/fuzzdb-project/fuzzdb)
9. [lcatro/PHP-webshell-Bypass-WAF](https://github.com/lcatro/PHP-webshell-Bypass-WAF)
10. [linuxsec/indoxploit-shell](https://github.com/linuxsec/indoxploit-shell)
11. [b374k/b374k](https://github.com/b374k/b374k)
12. [LuciferoO/webshell-collector](https://github.com/LuciferoO/webshell-collector)
13. [tanjiti/webshell-Sample](https://github.com/tanjiti/webshell-Sample)
14. [JoyChou93/webshell](https://github.com/JoyChou93/webshell)
15. [webshellpub/awsome-webshell](https://github.com/webshellpub/awsome-webshell)
16. [xypiie/webshell](https://github.com/xypiie/webshell)
17. [leett1/Programe/](https://github.com/leett1/Programe/)
18. [lhlsec/webshell](https://github.com/lhlsec/webshell)
19. [feihong-cs/JspMaster-Deprecated](https://github.com/feihong-cs/JspMaster-Deprecated)
20. [threedr3am/JSP-Webshells](https://github.com/threedr3am/JSP-Webshells)
21. [oneoneplus/webshell](https://github.com/oneoneplus/webshell)
22. [fr4nk404/Webshell-Collections](https://github.com/fr4nk404/Webshell-Collections)
23. [mattiasgeniar/php-exploit-scripts](https://github.com/mattiasgeniar/php-exploit-scripts)

### 白样本：
1. [WordPress/WordPress](https://github.com/WordPress/WordPress)
2. [yiisoft/yii2](https://github.com/yiisoft/yii2) 
3. [johnshen/PHPcms](https://github.com/johnshen/PHPcms)
4. [https://www.kashipara.com](https://www.kashipara.com/)
5. [joomla/joomla-cms](https://github.com/joomla/joomla-cms)
6. [laravel/laravel](https://github.com/laravel/laravel)
7. [learnstartup/4tweb](https://github.com/learnstartup/4tweb)
8. [phpmyadmin/phpmyadmin](https://github.com/phpmyadmin/phpmyadmin)
9. [rainrocka/xinhu](https://github.com/rainrocka/xinhu)
10. [octobercms/october](https://github.com/octobercms/october)
11. [alkacon/opencms-core](https://github.com/alkacon/opencms-core)
12. [craftcms/cms](https://github.com/craftcms/cms)
13. [croogo/croogo](https://github.com/croogo/croogo)
14. [doorgets/CMS](https://github.com/doorgets/CMS)
15. [smarty-php/smarty](https://github.com/smarty-php/smarty)
16. [source-trace/phpcms](https://github.com/source-trace/phpcms)
17. [symfony/symfony](https://github.com/symfony/symfony)
18. [typecho/typecho](https://github.com/typecho/typecho)
19. [leett1/Programe/](https://github.com/leett1/Programe/)
20. [rpeterclark/aspunit](https://github.com/rpeterclark/aspunit)
21. [dluxem/LiberumASP](https://github.com/dluxem/LiberumASP)
22. [aspLite/aspLite](https://github.com/aspLite/aspLite)
23. [coldstone/easyasp](https://github.com/coldstone/easyasp)
24. [amasad/sane](https://github.com/amasad/sane)
25. [sextondb/ClassicASPUnit](https://github.com/sextondb/ClassicASPUnit)
26. [ASP-Ajaxed/asp-ajaxed](https://github.com/ASP-Ajaxed/asp-ajaxed)
27. [https://www.codewithc.com](https://www.codewithc.com/)

### 综合数据集

处理后的综合数据集存放在 [Hugging Face](https://huggingface.co/datasets/c01dsnap/Webshell).

## 模型结构

程序会从指定的文件夹中读取指定类型的文件，计算这些文件的大小和熵值，以及通过 [nltk](https://www.nltk.org) 进行词分割。分割好的词传入 `tf.keras.layers.TextVectorization` 建立词库并完成向量化，然后传入 TextCNN 网络。文件的大小和熵值通过归一化处理后，传入一个二分类网络。

其中，TextCNN 网络的结构为输入层，嵌入层，3 个卷积核大小分别为 3、4、5 的卷积层，然后将 3 个卷积层的池化结果拼接后传入全连接层，插入 Dropout 层防止过拟合，最后传入输出层。二分类网络就是简单的 MLP 网络。最后将两个网络连接，获取最终的判断结果。

网络结构如下：
![](/img/WebshellCNN/WebshellCNN1.png)

## 结果评估

训练过程中的表现如下：
![](/img/WebshellCNN/WebshellCNN2.png)

模型评估结果如下：
![](/img/WebshellCNN/WebshellCNN3.png)
