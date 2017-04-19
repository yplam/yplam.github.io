---
title:  "关于《 Hands-On Machine Learning with Scikit-Learn and TensorFlow 》"
categories: MachineLearning
---

最近看了一本挺不错的书：[《 Hands-On Machine Learning with Scikit-Learn and TensorFlow 》](http://shop.oreilly.com/product/0636920052289.do)，
这本书分两部分：前半部分讲解了机器学习的一些基础知识，以及在 sklearn 上的实现；后半部分先介绍 Tensorflow，然后是深度学习的内容。虽然书的内容很明显
的分成两个部分，但看下去就会发现两部分的内容结合得很紧密，你需要不断的翻回前面去复习，在这个过程中也可以通过比对 sklearn 与 Tensorflow 对同一个模型的
不同实现来加深理解。

这本书还有一个很有特色的地方是在介绍深度网络的训练与调参的章节，会涉及一些非常新的内容，并且他的叙述方式可以让你明白为什么这么做，譬如关于梯度消失/爆炸问题：

* BackPropagation 算法中，越低层的学习速度越慢，甚至出现梯度过小导致权重值一直保持不变的情况
* 然后 2010 年，avier Glorot 与 Yoshua Bengio 发表的paper “Understanding the Difficulty of Training Deep Feedforward Neural Networks” 中提出了一些猜想的方式，结合 logistic sigmoid 激活函数与均值为 0，标准差为 1 的正态分布来对权重进行初始化；然而效果并不好，因为...
* Xavier 与 He Initialization 提出了一个新的初始化方法，根据输入输出神经元数量不同选择不同的标准差，在 Tensorflow 中可以用这个方法实现...
* Glorot 与 Bengio 还提出了 ReLU 激活函数，但是有 dying ReLU 问题，可以用 leaky ReLU，后来 15 年又有人提出 ELU，在 Tensorflow 中可以这样使用...
* 用 Batch Normalization 来更进一步的解决这个问题，在 Tensorflow 中这样实现...

总之，跟随作者的这种方式，会对 DNN 的一些概念有更清晰的了解。

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
