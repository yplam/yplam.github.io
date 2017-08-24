---
title:  "《 Hands-On Machine Learning with Scikit-Learn and TensorFlow 》笔记"
categories: MachineLearning
---

最近看了一本挺不错的新书：[《 Hands-On Machine Learning with Scikit-Learn and TensorFlow 》](http://shop.oreilly.com/product/0636920052289.do)，在此分享一下。

这本书分两部分：前半部分讲解了机器学习的一些基础知识，以及在 sklearn 上的实现；后半部分先介绍 Tensorflow，然后是深度学习的内容，最后就是在TensorFlow上的实现。
书上关于数学的内容都写得比较简略，所以需要边看边翻相关基础知识。而后半部分关于 TensorFlow 的内容就比较深入，介绍了不少作者在工程实践上的技巧。

个人认为第十一章《 Training Deep Neural Nets 》算是本书精华，譬如关于梯度消失/爆炸问题：

* BackPropagation 算法中，越低层的学习速度越慢，甚至出现梯度过小导致权重值一直保持不变的情况
* 然后 2010 年，avier Glorot 与 Yoshua Bengio 发表的paper “Understanding the Difficulty of Training Deep Feedforward Neural Networks” 中提出了一些猜想的方式，结合 logistic sigmoid 激活函数与均值为 0，标准差为 1 的正态分布来对权重进行初始化；然而效果并不好，因为...
* Xavier 与 He Initialization 提出了一个新的初始化方法，根据输入输出神经元数量不同选择不同的标准差，在 Tensorflow 中可以用这个方法实现...
* Glorot 与 Bengio 还提出了 ReLU 激活函数，但是有 dying ReLU 问题，可以用 leaky ReLU，后来 15 年又有人提出 ELU，在 Tensorflow 中可以这样使用...
* 用 Batch Normalization 来更进一步的解决这个问题，在 Tensorflow 中这样实现...

还有关于模型训练问题，譬如在TensorFlow中怎么加载预训练的模型，冻结低层参数来训练，缓存其输出；各种优化算法，各种防 Overfitting 的方法。

总之，跟随作者的这种方式，会对 DNN 的一些概念有更清晰的了解（至少对我这个门外汉而言是这样）。

下面截取附录中关于机器学习项目的一般步骤。

主要分为 8 步：

1. Frame the problem and look at the big picture.
2. Get the data.
3. Explore the data to gain insights.
4. Prepare the data to better expose the underlying data patterns to Machine Learning algorithms.
5. Explore many different models and short-list the best ones.
6. Fine-tune your models and combine them into a great solution.
7. Present your solution.
8. Launch, monitor, and maintain your system.

书中对每一个步骤都有解释，推荐到 O'Reilly 上购买一份电子版（可以Google一下，有50%的优惠券）。

**目录：**（点击查看原图）

[<img src="{{ site.url }}/assets/2017/hands-on-machine-learning.png" width="1000" />]({{ site.url }}/assets/2017/hands-on-machine-learning.png)
