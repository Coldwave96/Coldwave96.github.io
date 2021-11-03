---
title: 机器学习实现恶意URL检测Part 2
date: 2020-04-20 20:17:02
categories:
- Program
- Python
tags:
- Machine Learning
- Malicious URL
---
## 简述

&emsp;&emsp;在原来[第一版恶意URL检测](https://github.com/Coldwave96/MaliciousURLs)的基础上，改进了数据处理方式，调整了代码结构，实现了[第二版的恶意URL检测](https://github.com/Coldwave96/MaliciousUrls_Part2)。

<!-- more -->

## 提取数据

&emsp;&emsp;这一版添加了从pcap数据包中提取url的代码。通过[scapy_http库](https://github.com/invernizzi/scapy-http)从pcap包中筛选http协议的报文，再从报文中截取url保存为满足条件的文件格式。

```Url
/html/device-id
/centreon/include/views/graphs/graphStatus/displayServiceStatus.php?session_id=%27%20or%201=1%20--%20/**&template_id=%27%20UNION%20ALL%20SELECT%201,2,3,4,5,CHAR(59,%2032,%2099,97,116,32,47,101,116,99,47,112,97,115,115,119,100,%2059),7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23%20--%20/**
/centreon/include/configuration/configObject/traps/GetXMLTrapsForVendor.php
/admin/backup/
```

## 处理数据

&emsp;&emsp;训练的样本数据也最好处理成上面那种数据格式，这样有利于最大化的处理数据，提取特征。

&emsp;&emsp;本次训练的样本来部分自于安全设备上收集的url以及互联网上的样本。只是为了验证本次实验代码的效率，所以样本覆盖面不是很广，在实验最后总结中还会再次提到这个问题。

## 数据预处理

&emsp;&emsp;数据预处理的方法还是处理文本数据最典型的ngram算法和TF-IDF模型。通过自定义切割长度的ngram算法将原始url进行切割。然后通过TF_IDF模型把不规律的文本字符串转换成规律的（[i，j]，weight）的矩阵。

&emsp;&emsp;这一次的升级在于不是直接将TF-IDF模型处理完的数据投入分类器做训练，而是首先通过kmeans算法做一次聚合。通过指定聚合维度，对数据进行一次降维。降维后的数据在投入分类器，这样能够大幅度增加预测的准确度。

## 建模

&emsp;&emsp;建模算法可选择多种，本次样本数据集较小，采用的是svm和逻辑回归两种，包括NB算法在内的多种方法也有着不错的效果，更具实际情况可以灵活选用分类器算法。

```Python
# 读取数据
good_query_list = load_files(good_dir)
bad_query_list = load_files(bad_dir)

# 整合数据
data = [good_query_list, bad_query_list]

# 打标记
good_y = [0 for i in range(len(data[0]))]
bad_y = [1 for i in range(len(data[1]))]

# 分割数据集
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# 训练
self.classifier.fit(X_train, y_train)
```

## 预测

&emsp;&emsp;预测的数据可通过自定义文件位置实现，预测结果可以通过多种形式返回，目前是将正常结果和恶意结果前几条返回。

## 后记

* 本次试验吸取其他人的经验，新定义了一个输出方法，该方法会输出每一个步骤的具体时间，精确到秒级别，便于后续的测试调优。

```Python
def printT(word):
    a = time.strftime('%Y-%m-%d %H:%M:%S: ', time.localtime(time.time()))
    print(a + str(word))
```

* 本次实验相较于上一次，在训练之前多了一次kmeans聚类过程，大大提高了预测准确性。

* 但是增加的kmeans聚合的步骤也增加了训练的时间，实际测试百万级别的数据需要增加2个多小时的训练时间。

* 最后关于样本，前面也提到了本次样本只是在网上随便收集的。后续可以针对某个单位的流量针对性的训练，毕竟代码可以复用，只需要不同的样本即可。另外还有一个思路是，训练样本可以只包括白样本，训练出一个白样本模型。实战时所有在白样本模型之外的请求都默认为恶意请求，这样的模型安全性比较好。