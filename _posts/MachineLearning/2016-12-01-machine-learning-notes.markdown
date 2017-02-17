---
title:  "机器学习笔记"
categories: MachineLearning
---

机器学习是人工智能的一个分支。人工智能的研究是从以“推理”为重点到以“知识”为重点，再到以“学习”为重点，一条自然、清晰的脉络。显然，机器学习是实现人工智能的一个途径，即以机器学习为手段解决人工智能中的问题。机器学习在近30多年已发展为一门多领域交叉学科，涉及概率论、统计学、逼近论、凸分析、计算复杂性理论等多门学科。机器学习理论主要是设计和分析一些让计算机可以自动“学习”的算法。机器学习算法是一类从数据中自动分析获得规律，并利用规律对未知数据进行预测的算法。因为学习算法中涉及了大量的统计学理论，机器学习与推断统计学联系尤为密切，也被称为统计学习理论。[来源](https://zh.wikipedia.org/wiki/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0)

本文记录机器学习相关的概念与资料索引。

### 机器学习分类

机器学习可以分成下面几种类别：

* 监督学习从给定的训练数据集中学习出一个函数，当新的数据到来时，可以根据这个函数预测结果。监督学习的训练集要求是包括输入和输出，也可以说是特征和目标。训练集中的目标是由人标注的。常见的监督学习算法包括回归分析和统计分类。
* 无监督学习与监督学习相比，训练集没有人为标注的结果。常见的无监督学习算法有聚类。
* 半监督学习介于监督学习与无监督学习之间。
* 增强学习通过观察来学习做成如何的动作。每个动作都会对环境有所影响，学习对象根据观察到的周围环境的反馈来做出判断。


### 机器学习资料及读后感

* 李航《统计学习方法》，薄薄的一本书，内容很清晰，直接了当，但没有涉及数学基础的讲解，需要边看边翻数学书恶补基础，总的来说值得化时间认真看几遍。感知机、k近邻法、朴素贝叶斯法、决策树、逻辑斯谛回归与最大熵模型、支持向量机、提升方法、EM算法、隐马尔可夫模型和条件随机场。
* TOBY SEGARAN 《集体智慧编程》，应用类的书，主要是为了加深理解，并且避免连续看纯理论的书影响积极性。
* [Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/index.html)用一种比较简单的方式介绍神经网络、深度学习、卷积神经网络。特别是用了独立的一章来介绍反向传播算法，讲解以及代码实现都非常清晰。
* [Principles of training multi-layer neural network using backpropagation](http://galaxy.agh.edu.pl/~vlsi/AI/backp_t_en/backprop.html) 一个非常简单的关于反向传播算法图解。
* [colah's blog](http://colah.github.io/)，一位研究员的博客，写得很用心详细。内容涉及[神经网络(NN, Neural Networks)](http://colah.github.io/posts/2014-03-NN-Manifolds-Topology/)，[循环神经网络(RNN, Recurrent Neural Networks)](http://colah.github.io/posts/2015-08-Understanding-LSTMs/),[卷积神经网络(CNN, Convolutional Neural Networks)](http://colah.github.io/posts/2014-07-Conv-Nets-Modular/), [神经网络可视化](http://colah.github.io/posts/2014-10-Visualizing-MNIST/).
* [The Elements of Statistical Learning](http://statweb.stanford.edu/~tibs/ElemStatLearn/),正在看的一本书，主要涉及机器学习相关数学基础，可以作为《统计学习方法》的补充，未完待续。 
* [CS231n课程笔记翻译：反向传播笔记](https://zhuanlan.zhihu.com/p/21407711),对反向传播解析得比较仔细。
* [深度学习论文阅读路线图](https://github.com/songrotek/Deep-Learning-Papers-Reading-Roadmap)

