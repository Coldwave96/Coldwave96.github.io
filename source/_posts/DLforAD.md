---
title: Deep Learning for Anomaly Detection - A Review
date: 2023-03-21 16:36:58
categories:
- Essay
- Notes
tags:
- Anomaly Detection
- Deep Learning
---
## Info

* 名称：Deep Learning for Anomaly Detection: A Review
* 作者：
	* GUANSONG PANG, University of Adelaide
	* CHUNHUA SHEN, University of Adelaide
	* LONGBING CAO, University of Technology Sydney
	* ANTON VAN DEN HENGEL, University of Adelaide
* 原文链接：[arxiv](https://arxiv.org/abs/2007.02500)

<!-- more -->

## Intro

* 异常检测（Anomaly / Outlier / Novelty detection）在过去的几十年里都是热门研究话题，但是仍然存在许多复杂的问题和挑战需要更进一步的研究。
* 这篇论文主要介绍了基于深度学习的异常检测技术，涵盖了3大类，11个小类的理论方法。

## Contributions

1. 问题本质和挑战。作者提到了异常检测中遇到的特殊问题的复杂度以及由此造成的许多未解决的挑战。
2. 问题分类和总结。本文将现有的深度学习异常检测方法归总为3中理论框架：深度学习普遍特征提取，正常样本的表示，端到端异常分数学习。所有的方法从11个不同的建模层面进行的分类。
3. 深度解析其他的论文研究内容。
4. 未来的机遇与挑战。
5. 源代码及数据集。


## Paragraph
* 由于异常检测问题本身的特质导致的复杂性：
    - 未知性
    - 异常与异常之间就有着不同的特征
    - 异常样本数量极少导致的黑白样本比例极不均衡
    - 异常类型的多种多样
        - 点异常
        - 条件异常
        - 组异常

* 深度学习异常检测面临的挑战
    - CH1：低recall rate（样本中的正例多少被预测正确， TP/TP+FN）
    - CH2：面对高维数据或者相互不独立数据表现一般，如何降维以及降维后如何保证原有信息完整也是挑战
    - CH3：由于数据量的有限，如何提高数据使用的有效性。监督学习需要大量的有标签数据，非监督学习依赖于对于异常分布的正确假设，半监督学习是一种解决方向。另一种解决方向是`weakly-supervised anomaly detection`
    - CH4：许多弱监督/半监督算法抗噪能力不好
    - CH5：现有很多异常检测算法均针对点异常，面对上下文异常及组异常效果不好
    - CH6：模型可解释性不强，面对某些争议难以合理解释
<center>
    <img src="/img/DLforAD/1.png" width="800">
</center>

* 深度学习异常检测方法分类
<center>
    <img src="/img/DLforAD/2.png" width="850">
</center>

* 深度学习特征提取
    - 直接运用流行的深度学习模型AlexNet【1】，VGG【2】，ResNet【4】等提取低维度特征
    - 训练深度学习特征提取模型进行异常分数评估

* 正常样本表示
    - 通用正常特征学习，优化通用样本数据特征表示方法
        - Autoencoder（AE）
        - GAN-based 异常检测
        - 基于预测模型的特征学习方法，用之前的数据实例预测现在的数据实例
        - 自监督学习
    - 针对存在的已知异常特殊优化的异常评估模型
        - 基于距离的评估手段
        - 针对单类型的异常分类器
        - 基于聚类的评估手段
    - 端到端异常评分学习 - 不仅仅局限于已知异常，着重于基于深度神经网络直接学习评估异常分数
        - 排名模型
            - 设计基于逻辑回归损失函数的异常评分模型【5】
            - 先验模型：已知数据集的分布特性
        - 概率模型 - 通过最大化在训练集中事件的可能性来学习异常评分
        - 端到端单种类分类器
            - 对抗学习单种类分类器（adversarially learned one-class classification，ALOCC）【6】

## Comments

1. 异常检测方法根据检测环境的不同有比较大的差距。理想情况下需要先了解实际环境中的异常种类、分布等信息，然后对症下药，选择合适的特征工程以及检测算法。但是大多数情况下均不满足这些条件，给检测工作带来许多难题，最终导致模型检测结果不尽如人意。
2. 深度学习可以针对性的解决部分异常检测面临的挑战，但是也需要针对具体问题灵活选择合适算法。

## References
1. [Alex Krizhevsky, Ilya Sutskever, and Geoffrey E Hinton. 2012. Imagenet classification with deep convolutional neural networks.](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#3e5880a9bd784529ad5022b751a168ba)
2. [Karen Simonyan and Andrew Zisserman. 2015. Very deep convolutional networks for large-scale image recognition. In ICLR.](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#a854d669d0e74fcf9241cdbab8d2ba3f)
3. [Radu Tudor Ionescu, Sorina Smeureanu, Bogdan Alexe, and Marius Popescu. 2017. Unmasking the abnormal events in video. In ICCV. 2895–2903.](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#c42b25debe154c99ac44769a22218463)
4. [Kaiming He, Xiangyu Zhang, Shaoqing Ren, and Jian Sun. 2016. Deep residual learning for image recognition. In CVPR. 770–778.](https://openaccess.thecvf.com/content_cvpr_2016/html/He_Deep_Residual_Learning_CVPR_2016_paper.html)
5. [Guansong Pang, Cheng Yan, Chunhua Shen, Anton van den Hengel, and Xiao Bai. 2020. Self-trained Deep Ordinal Regression for End-to-End Video Anomaly Detection. In CVPR. 12173–12182.](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#b46593aa01974e8aa73a0f89fba37aca)
6. [Mohammad Sabokrou, Mohammad Khalooei, Mahmood Fathy, and Ehsan Adeli. 2018. Adversarially learned one-class classifier for novelty detection. In CVPR. 3379–3388.](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#9075f53359e1410bbd44f59325c117de)
7. [Guansong Pang, Chunhua Shen, Huidong Jin, and Anton van den Hengel. 2019. Deep Weakly-supervised Anomaly Detection. arXiv preprint:1910.13601 (2019).](https://www.notion.so/Deep-Learning-for-Anomaly-Detection-A-Review-27ba856b85f147f1b3cb114a93848f63?pvs=4#78435f94252346e1b2e1295acb851f75)

## Related Materials
* 算法列表
<center>
    <img src="/img/DLforAD/3.png" width="850">
</center>
<center>
    <img src="/img/DLforAD/4.png" width="850">
</center>
* [数据集列表](https://git.io/JTs93)
<center>
    <img src="/img/DLforAD/5.png" width="850">
</center>