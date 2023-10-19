---
title: 机器学习实现恶意URL检测
date: 2019-11-08 14:17:02
categories:
- Program
- Python
tags:
- Machine Learning
- Malicious URL
---
## 简述

作为正在开发的态势感知产品中重要的一部分，初步实现了[恶意URL的AI检测功能模块](https://github.com/Coldwave96/MaliciousURLs)。该模块将集成到大数据平台的检测引擎中去，将对平台推过来的每一条URL进行检测，检测结果实时反馈回平台。

通过机器学习检测Web请求包中的URL/URI判断是否为恶意URL，暂时不做细分，只做笼统的考量，后期可以通过训练集中添加细分的样本数据重新构建能够细分详细攻击类型的模型，比如分辨SQL注入攻击，XSS攻击，恶意域名等。

模型生成完毕之后，将供大平台直接调用模型对每一个待检测数据包进行判断，生成基础事件。

关于不同类型的攻击模型，生成模型的代码基本都一样，主要差别在于使用的样本即训练集不一样，本次训练使用的是恶意URL的样本集，里面包含了SQL注入、恶意域名等。所以要想生成高精度的攻击检测模型，关键在于使用好的专用训练集。

<!-- more -->

## 处理数据

通过探针抓取到的数据会被解析成应用层json格式数据，通过分类处理不同method剥离出URI/URL作为待处理数据（P.S.对于GET数据直接将URI/URL作为分析对象，对于POST请求将URI/URL后面拼接上body中的数据字段作为分析对象）。

![](/img/MaliciousURLs/MaliciousURLs1.png)

## 数据预处理及建模

处理对象为分离后的URI/URL，如下图所示：

![](/img/MaliciousURLs/MaliciousURLs2.png)

然后读取数据，并给黑白样本打上标签，切割文本，将文本数据转化为矩阵向量，通过TF-IDF方法提取特征，训练模型采取逻辑回归算法实现。

```Python
 def __init__(self):
        #读取数据
        good_query_list = self.get_query_list('goodqueries.txt')
        bad_query_list = self.get_query_list('badqueries.txt')

        #给黑、白数据分别打标签
        good_y = [0 for i in range(0, len(good_query_list))]
        bad_y = [1 for i in range(0, len(bad_query_list))]

        queries = good_query_list + bad_query_list
        y = good_y + bad_y

        #将原始文本数据分割转化成向量
        self.vectorizer = TfidfVectorizer(tokenizer=self.get_ngrams)

        #把文本字符串转化成（[i,j],Tfidf值）矩阵X
        X = self.vectorizer.fit_transform(queries)

        #分割训练数据（建立模型）和测试数据（测试模型准确度）
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=20, random_state=42)

        #定义模型训练方法（逻辑回归）
        self.lgs = PMMLPipeline([('LogisticModer', LogisticRegression(solver='liblinear'))])

        #训练模型
        self.lgs.fit(X_train, y_train)

        #测试模型准确度
        print('模型准确度:{}'.format(self.lgs.score(X_test, y_test)))

        sklearn2pmml(self.lgs, '.\lgs.pmml', with_repr=True)
```

分割字符串通过自己定义的`get_ngrams(self, query)`方法实现：

```Python
def get_ngrams(self, query):
    tempQuery = str(query)
    ngrams = []
    for i in range(0, len(tempQuery)-3):
        ngrams.append(tempQuery[i:i+3])
    return ngrams
```

## 预测

预测的数据格式为：

![](/img/MaliciousURLs/MaliciousURLs3.png)

预测通过`predict(self, newQueries)`方法实现：

```Python
def predict(self, newQueries):
    newQueries = [urllib.parse.unquote(url) for url in newQueries]
    X_predict = self.vectorizer.transform(newQueries)
    res = self.lgs.predict(X_predict)
    res_list = []
    for q,r in zip(newQueries, res):
        tmp = '正常请求' if r == 0 else '恶意请求'
        q_entity = html.escape(q)
        res_list.append({'url':q_entity, 'res':tmp})
    print("预测的结果列表:{}".format(str(res_list)))
    return res_list
```

预测的结果保存为json格式方便推回给大数据平台处理：

![](/img/MaliciousURLs/MaliciousURLs4.png)

## 产品化工作

为了便于产品化的实现，设计了两种方法思路。刚开始尝试将整个数据的预处理，训练及建模过程写入一个类，然后将整个类序列化，保存为.pickle格式的文件，然后供大数据平台调用。后来为了增加数据处理的效率，仅仅将训练好的模型保存为.pmml格式的文件供大数据平台调用。

## 可能的改进

* 处理结果细化，将恶意请求再次具体细分为SQL注入，XSS攻击，恶意域名等。

* 在机器学习算法前提下，特征处理方法的改进，寻求优化TF-IDF方法方案。目前采用的是仅仅使用TF-IDF算法对数据进行预处理，后面可以考虑采用词袋+TF-IDF算法，或者结合TF-IDF算法将URL中不重要的部分进行泛化处理。

* 跳出机器学习算法，省去特征处理的过程，改用深度学习算法，构建神经网络（需要花费大量时间不断计算，调优）。

* 目前可以将同一个IP的一个时间段之内的URL请求同时进行预测，来判断该IP是否实施攻击，也可以只检测单条URL判断是否为恶意请求。未来可改进为不剥离请求包中IP标签，对一段时间内所有数据进行检测，分出有攻击行为的IP上报。
