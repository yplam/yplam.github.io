---
title:  "[翻译]用户行为预测：使用机器学习技术预测付费用户"
categories: MachineLearning
---

**注**：本文为 [Predicting customer behavior: How we use machine learning to identify paying customers before they subscribe](http://www.strong.io/blog/predicting-customer-behavior-machine-learning-to-identify-paying-customers) 的简单翻译版本，详细内容请阅读原文。

本案例为 strong.io 公司的一位创业公司客户，该客户开发了一个类SaaS软件，但在经过多年的稳固发展后，他们的每月收入增长却开始变得很缓慢。他们曾经尝试改善他们的产品，但效果可能不好，因此现在将重点放在客户获取上面。
该产品现在每月有超过3000个试用下载，但只有大约3%的用户转化为付费版。

为了弄清是什么原因，他们开始做一个试验：雇佣一些人员，用整整一个月时间，打电话或者email给那些下载了试用版的用户，问问他们对产品的看法，并且帮助他们解决问题。这样一个月过去后，他们的转化率从3.7%上升到5.4%；同时，他们知道了其中很多用户根本对购买付费版没兴趣，他们仅仅是想使用一下免费版里面带的一些工具--他们并不是目标客户！

他们希望 strong.io 通过技术手段将潜在的付费用户区分出来，然后让客户支持团队将精力集中在该潜在客户上。


### 解决方案

1、记录完整的用户行为信息，每一个用户都会记录维度为70的数据。

2、测试各种机器学习方法。使用 AUC (Area Under the Curve) 方法进行评价；使用一年的数据进行模型训练，使用随后半年的数据进行测试； 其中 Logistic 模型最好可以达到 78% AUC，随机森林模型可以达到 80% AUC，梯度提升模型可以达到 93% AUC。最后，选用了RNN， 可以达到 95% AUC。

3、生成每个试用用户的转化概率，集成到CRM中

4、最终，该公司的转化率上升并且稳定在7.4%.


